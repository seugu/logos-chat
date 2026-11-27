# GroupV3: getting every commit to every member

A GroupV3 member finalizes a commit on a timer. It defers every commit it sees for the current
epoch and, `commit_finalize_timeout` (120 s) after the first one, applies the one with the lowest
fingerprint. Members agree only if each of them has seen the same commits by then. A member that
misses the winning commit applies another one, and the group forks.

## Previous problems

In simulation (chat-sim, `grow-waku-loss-*-groupv3`, three members adding at once), almost every
run forked. In all 20 seeds traced, the first fork came about the same way. The lowest commit was
lost at one or two members, and it did not come back to them within 120 s. Three things kept it
away:

- **An acknowledgement needs a message to ride on.** SDS has no ACK message. A receiver
  acknowledges only through the history and bloom filter of the next message it sends. A member
  with nothing to send syncs only after 30 s of quiet, and every message it receives resets that
  wait. So in a busy group most members never sent anything, and senders never heard from them.
- **One acknowledgement stops the resends.** A sender stops resending once any single member names
  the message. That member is usually not the one that missed it.
- **A member never asked for a message it lacked.** Syncs from other members did name the lost
  commit. SDS saw this, but passed it only to SDS-R, which GroupV3 turns off. The member noticed
  the gap only when a later content message named the commit, typically about 180 s later. By then
  its timer had already fired.

## What this change does

- **Fetch what a sync shows missing.** sds-rs reports `Event::SyncNamesMissing` when a sync names
  messages this member lacks, and GroupV3 fetches them from the store, the same way it fetches what
  a buffered message is missing. A lost commit is now fetched as soon as any member syncs.
- **Acknowledge soon after receiving.** With `ack_sync_max_delay_ms` (GroupV3 sets 5 s), a member
  syncs within that delay of receiving a message, even if it has nothing to send. One sync covers
  everything received before it, and a sync never triggers another. Senders hear from quiet
  members, and the syncs that carry these ACKs also tell other members what they missed.
- **Configurable finalize timeout.** `Core::set_group_v3_config(GroupV3Config { commit_finalize_timeout })`
  replaces the hard-coded 120 s. The default is unchanged. Every member of a group should use the
  same value.

## Why it works

The lost commit is the problem, not the timer. Once a member fetches the commit within the
timeout, it sees the same set of commits as everyone else and finalizes the same one. Before, only
a content message from a later round brought the commit back. Now any member's sync does, a few
seconds after the loss. The ACK syncs make those syncs frequent even when few members commit.

## Results

The runs use chat-sim's own GroupV3 scenarios and tools, with 10 runs per `grow` scenario and 2 per
`50-epochs` scenario. `master` is chat-sim as is. "This change" is the same chat-sim with logos-chat
pinned to this change backported onto `a74ad35`, the revision chat-sim runs. The count is of runs
that ended as one group: a single main line, no side branches, and no member behind.

| Scenario | `master` | This change |
|---|---|---|
| `grow-waku-loss-0-groupv3` | 10/10 | 10/10 |
| `grow-waku-loss-0.1-groupv3` | 4/10 | **10/10** |
| `grow-waku-loss-0.5-groupv3` | 1/10 | **10/10** |
| `grow-waku-loss-1-groupv3` | 0/10 | **10/10** |
| `grow-waku-loss-5-groupv3` | 0/10 | **10/10** |
| `50-epochs-by-saro-waku-loss-1-groupv3` | 1/2 | **2/2** |
| `50-epochs-by-saro-waku-loss-5-groupv3` | 0/2 | **2/2** |

- **grow** has a 200-member pool, and each round 1 to 3 members commit at the same instant, aiming
  for 50 epochs. On `master`, the trunk at 5% loss reaches epoch 23 on average, with about 5 side
  branches. With this change it reaches 50 in every run.
- **50-epochs** has one committer, so it never forked. On `master`, members fell behind at 1% and
  5% loss. With this change none do, and 50 epochs at 5% loss take 6254 s of virtual time instead
  of 7849 s.
- `sim matrix` gives the verdict `later`: no run forked earlier with this change. At 0.5%, 1% and
  5% loss the difference is beyond chance (sign test p ≤ 0.004).
- A run is deterministic on Linux. The numbers were reproduced to the second on a second machine.

To reproduce them, or to run your own variants (loss, delays, number of adders, partitions), see
`DELIVERY-FIX.md` on [seugu/chat-sim `sds-reliability-fixes`](https://github.com/seugu/chat-sim/tree/sds-reliability-fixes).
The backports it runs are
[seugu/logos-chat `sds-reliability-fixes-a74ad35`](https://github.com/seugu/logos-chat/tree/sds-reliability-fixes-a74ad35)
and [seugu/sds-rs `sync-reports-missing-648283e`](https://github.com/seugu/sds-rs/tree/sync-reports-missing-648283e).
This change on `main` has not been run in chat-sim, which still pins `a74ad35`.

## Limits

- Agreement still rests on the timer. A commit that cannot be fetched within the timeout still
  forks the group. This change makes that rare. It does not make it impossible.
- An SDS acknowledgement means "received", not "applied". A message can be acknowledged while it
  still waits in the causal buffer.
- More traffic: in earlier experiments the ACK syncs about doubled the bytes per epoch.
