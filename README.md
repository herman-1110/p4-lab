# Week 5 lab — Programmable Data Planes (P4)

Two P4 software switches (BMv2) in a Mininet network, both running `tunnel.p4`. You'll
watch a P4 pipeline forward IPv4, change its tables at run time, then finish a custom
tunnel header — and see what happens when the parser, the ingress and the deparser
disagree.

**The full step-by-step instructions are in the Lab tab of Week 5 on the course website.**

## Start

Make your own copy first, so you can save your work:

1. At the top of this repository's page, click **Use this template → Create a new repository**.
   Create it under your own account (public or private).
2. In **your** new repository: **Code → Codespaces → Create codespace on main**.

The first start takes a minute or two while the lab image downloads. Then, in the terminal:

```bash
python3 topo.py --id NNNN
```

That compiles `tunnel.p4`, starts the network, fills the switches' tables and opens the
`mininet>` prompt. Type `exit` to stop it; run it again after every change to `tunnel.p4`.

## Assignment

After the lab, the Week 5 assignment continues it: **put a hop limit on the tunnel**. The
instructions are at the end of the Week 5 Lab tab on the course website. Work on a branch,
write `assignment/REPORT.md`, save your evidence in `assignment/evidence/`, and submit your
pull request's link on Spectrum. Every push is checked automatically: the **Compile tunnel.p4**
check must be green.

`python3 topo.py --id NNNN --s2-program tunnel_v1.p4` runs s2 with a different program, for the
assignment's mixed-rollout experiment.

## Files

| File | What it is |
|---|---|
| `tunnel.p4` | The P4 program both switches run. You edit this. |
| `topo.py` | Builds the network: h1 — s1 — s2 — h2, h3. |
| `s1-commands.txt`, `s2-commands.txt` | Table entries loaded into each switch at start. |
| `send.py` | Sends one test packet, optionally tunnelled (run it on h1). |
| `receive.py` | Started automatically on h2 and h3; output in `logs/h2.txt`, `logs/h3.txt`. |
| `assignment/REPORT.md` | The assignment's report template. |
| `p4cli` | Sends one command to a switch's runtime CLI, e.g. `./p4cli s2 table_dump MyIngress.ipv4_lpm`. |
| `logs/s1.txt`, `logs/s2.txt` | What each switch did with every packet: parser states, tables, actions. |

## If something goes wrong

- **Mininet complains that interfaces already exist** (after a crash): run `mn -c`, then `python3 topo.py --id NNNN` again.
- **Compilation failed**: the message gives the line number in `tunnel.p4`. Check for a missing `;` or `}`.
- **VS Code shows "cannot open source file core.p4"** (or `v1model.p4`) on `tunnel.p4`: that's
  VS Code's C checker, not the P4 compiler — ignore it. The errors that matter come from
  `python3 topo.py`.
- **The codespace stopped**: codespaces stop after 30 minutes idle. Restart it from github.com/codespaces; your edits are kept.
- **Save your work**: commit and push from the codespace (Source Control panel, or `git` in the
  terminal) — it goes to your own repository.
- **When you've finished the lab**, delete the codespace at github.com/codespaces so it stops using your storage quota.

## Week 5 lab notes (student ID ending 3501)

### Wrap-up answers
1. The two more things that I can change without recompiling is through the table entries where I can add, delete or modify routes, ports, tunnel exits and a table's default action such as making the ipv4_lpm's default forward instead of drop. Then the counters, registers, meters, multicast groups and mirroring sessions too where a program that has them. For a new program, a new header or parser state like how I did in Step 4, a new table, match field or action like matching on UDP port or decrementing a hop limit and any change to an apply block's magic like in Step 5.

2. This is because the ingress still applied only ipv4_lpm is keyed on the IPv4 destination 10.1.0.3 so dst_id 2 was never looked at while the compiler had even removed myTunnel_exact. Now, the parser extracted the tunnel header making it valid and the deparser emits every valid header so nothing invalidates it, and it left tunnelled.

3. The tunnelled packet ran myTunnel_forward which only sets the egress port and never touches the IPv4's TTL so it kept send.py's 64 TTL. The ping replies ran ipv4_forward on s2 and then s1: 64 -> 63 -> 62

4. The compiler first caught is that it warned myTunnel_exact was unused in Step 1 to 4 and it rejects syntax and type errors. It missed forwarding by IPv4 instead of the tunnel is Step 4, and the missing emit in Step 6. The compiler checks one program alone, only packets sent end to end, read in the logs, show whether parser, ingress, deparser and hosts agree.