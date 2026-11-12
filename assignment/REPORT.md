# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **3501**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

- The entry that I have changed is on s2's myTunnel_exact entry 2 (key 3) from 3 to 1, and the commands to make the loop is in the loop-before.txt which are:
     1. sh ./p4cli s2 table_dump MyIngress.myTunnel_exact
     2. sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
     3. h1 python3 send.py 10.1.0.3 "loop" --tunnel 3
     4. sh grep -c "Processing packet received" logs/s1.txt logs/s2.txt
     5. sh sleep 5; grep -c "Processing packet received" logs/s1.txt logs/s2.txt
- The counts from the loop-before.txt are 675/673, then 1439/1441 five seconds later
- For the decrement of TTL, it doesnt stop an IPv4 loop because ipv4_forward subtracts 1 but never checks it, and bit<8> wraps 0 to 255, so if the TTL reaches 0, it will not stop it will still keep going on and on because it will go from 0 to 255 again.

## 2. Design: the hop limit

- For the field size, I would go for 8 bits hop limit since 0-255 is far more than needed and the longest real path here is 2 switches. IPv4's TTL and IPv6's hop limit are also 8 bits. One byte is the smallest step because P4 headers must be the whole bytes and header grows from 4 to 5 bytes so each packet costs only 1 extra byte. As for where I would put it, is after the *dst_id*. This is due to *proto_id* and *dst_id* keep their position at bytes 0-1 and 2-3 so an old switch still reads them correctly and putting the field first would shift both old fields, taking bits from *dst_id* would shrink the exit numbers and break old switches’ lookups. Next is the known cost, which I’ll examine in C5 where an old switch still thinks IPv4 starts after *dst_id*, so it misreads the IPv4 header by one byte.

- For the starting value, it will be 8 which set by the send.py --hops. With “drop at 0”, a packet starting at 8 can cross exactly 8 switches. h1 to h2 crosses 2, so 2 is the minimum. --hops 1 is dropped at s2. 8 gives four times the headroom if the network grows, without changing the hosts. It’s still small enough that a loop dies fast. In C1’s loop, the packet makes 8 crossings and is dropped on the 9th pass, instead of looping forever.

- And where it's checked, is at the ingress, tunnel branch and befire the *myTunnel_exact.apply()*. Every switch that forwards a tunnelled packet runs this branch so every hop counts and checking before the lookup means a packet with no hops left is dropped whatever the table say so a bad entry like C1's cant keep it alive. In v1model, a drop in ingress happens before the packet is queued, so no queue space is wasted where plain IPv4 is untouched, so the ping in C4 still works.

- So when it runs out is when arriving with 0, otherwise it will subtract 1 and forward. 0 means "no crossing to this switch" which matches the field's meaning. When it checks before subtracting, the value never wraps to 255, which is the IPv4 TTL bug I found in C1. A normal delivery arrives with 6 which is visible in logs/h2.txt but the trade off is that drop is silent unlike the IPv4's ICMP "Time Exceeded" that nothing tells the sender.

- As for other things like the parser (extract) now reads 5 bytes and no code is changed because it follows the header definition, deparser (emit) writes the decremented value back for the next switch and hosts (send and receive py) both get ByteField("hop_limit", 8). Otherwise they’d misplace IPv4 by a byte, like how we managed to know in Step 6’s lesson.


## 3. Implementation

- The changes in tunnel.p4 is that I added an 8-bit hop_limit at the end of the tunnel header, which grows from 4 to 5 bytes. Then, also added a switch that drops a tunnelled packet that arrives with 0. Otherwise it subtracts 1 and forwards as before.

- For send.py, I added the new field and a --hops option (default 8).

- For receive.py, I added the new field, and it prints the hop limit that arrives.

- What the diff doesn't show are the parser and deparser weren’t edited, but they now read and write 5 bytes automatically. Next, is the check comes before the subtraction, so the value can’t wrap from 0 to 255 like the TTL did and the hosts must define the header exactly like the switches, or packets get misread, as in Step 6. Lastly, no table entries changed, and the IPv4 path is untouched while the import-line comments only hide false warnings in the editor.

## 4. Evidence: it works

**Delivery** ([evidence/delivery.txt](evidence/delivery.txt))

```
Received (tunnelled, dst_id=2, hop_limit=6) from 10.1.0.1 to 10.1.0.3, TTL 64: 'to exit 2'
Received (tunnelled, dst_id=3, hop_limit=6) from 10.1.0.1 to 10.1.0.2, TTL 64: 'to exit 3'
PING 10.1.0.3 (10.1.0.3) 56(84) bytes of data.
--- 10.1.0.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 6.530/7.621/8.712/1.091 ms
```

- Both exits got their packet, the hop limit arrived as 6 (8 minus 2 switches), and plain IPv4 still works

**The loop now ends** ([evidence/loop-after.txt](evidence/loop-after.txt))

```
before sending:  logs/s1.txt:972   logs/s2.txt:968
1 s after:       logs/s1.txt:977   logs/s2.txt:972
5 s later:       logs/s1.txt:977   logs/s2.txt:972
```

- +5 at s1 and +4 at s2 is 9 passes, 8 crossings and a drop on the 9th, which matches a starting value of 8 while we compare to the loop-before.txt C1's counts, which never stopped

## 5. Evaluation

| Option | How it works | What an old switch does | Trade-off |
| --- | --- | --- | --- |
| New EtherType (say 0x1213) | The new format gets its own type; new switches parse both | Doesn't recognise the type, stops parsing, drops it | Fails cleanly and never corrupts, but every switch on a path must be upgraded first |
| Next-header signal | Keep the old 4 bytes first; a new `proto_id` value means "hop limit, then IPv4" | Forwards by `dst_id`, parses no IPv4, passes the rest untouched | Mixed paths keep working; old switches just don't count hops |
| Version field | A version or length field tells formats apart | Can skip or reject formats it doesn't know | Works only if v1 had one; ours didn't, so add it now |
| Append the field (my design) | `hop_limit` added after `dst_id`, same EtherType | Forwards by `dst_id`, but reads IPv4 one byte early; its checksum step corrupted the protocol byte (`11` → `0d`) | No extra setup, but packets on mixed paths arrive damaged, and only new switches count hops |

### Loop Protection data plane vs control plane
- The hop limit ends any loop at run time, whatever caused it: a manual edit like C1's, a controller bug, a transient loop mid-update. But only after the packet has looped, silently, and only switches running the new program count.
- A controller-side path check can refuse a looping configuration before any packet suffers, and say where the loop is. But it sees only what the controller installed: not edits behind its back (C1's p4cli), switches out of sync, or data-plane bugs.
- Real networks use both: prevention above, a safety net below, like routing protocols plus IP's TTL.

### The cost at line rate
- One more header byte per packet: about 1.6% of a 64-byte frame, under 0.1% of a 1,500-byte one.
- One more field in the parser and the packet header vector, plus a compare and a subtract before the table lookup. That dependency can cost a pipeline stage.
- On PISA hardware the program either fits and runs at line rate or doesn't compile: the cost is resources, not speed. A 5-byte header also leaves IPv4 at an odd offset.

## AI use

**Tool:** Claude (Anthropic's AI assistant), used in a chat while I worked through the lab and this assignment.

**What Claude did:**
- Wrote a step-by-step runbook for the lab and the assignment, with the commands filled in for my addresses (10.1.0.x).
- Suggested the hop-limit design I used: an 8-bit `hop_limit` after `dst_id`, a starting value of 8, and a check in the ingress tunnel branch that drops a packet arriving with 0 and otherwise subtracts 1 before `myTunnel_exact`. It also listed the reasons for each choice.
- Gave the code changes for `tunnel.p4`, `send.py` (the `--hops` option) and `receive.py`, and checked my `send.py` and `receive.py` by running them on test packets. It could not compile the P4 program; `topo.py` and the Compile tunnel.p4 check did that.
- Helped me fix two problems: false editor warnings on the scapy imports, and a broken import line in `receive.py` that would have stopped it printing anything.
- Explained the BMv2 logs, why the IPv4 TTL wraps from 0 to 255, my mixed-rollout capture byte by byte, and the pass counts to expect.
- Gave points and layouts for sections 3–5, including the rollout-options table in 5b.

**What I did:**
- Ran every command and experiment in my own codespace; all the evidence files are my own output.
- Chose to use the suggested design, and checked my results against it: 9 passes with both switches upgraded, 17 in the mixed rollout.
- Wrote the report in my own words from Claude's points. 

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
