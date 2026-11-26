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

These are chat-sim runs on `grow-waku-loss-{1,5}-groupv3`: 200-member pool, three adders per round,
with extra per-packet jitter of 0 or 2 s. "Fork-free" means the whole run ended as one group.

| Mode | Fork-free runs | 1-add round at 5% loss | MB per epoch at 1% loss |
|---|---|---|---|
| Before | 0 / 40 | 121.5 s | 2.0 |
| Fetch on sync only | 40 / 40 | 155–157 s | 2.6 |
| Fetch on sync + ACK sync (this change) | 39 / 40 | 122–125 s | 3.7–4.0 |
| ACK sync only | 0 / 40 | 121.5 s | 2.6–3.2 |

- The ACK sync alone does not help. Faster ACKs only stop resends sooner. The fetch on sync is what
  removes the forks.
- The ACK sync keeps rounds short under loss, cutting a 1-add round from about 155 s to about
  122 s at 5% loss. Before, only 5% of commits were ever acknowledged by every member. Now
  97–100% are. At 1% loss the median time to that is about 7 s. At 5% loss it is about 2 min,
  because a lost ACK sync waits for the member's next message.
- A wider sweep of the fetch on sync alone was 149 of 150 runs fork-free, over 0.5–5% loss and
  0–10 s jitter.
- The remaining fork is a different case. A member joined through a commit that then lost, and
  waited on dependencies that never arrived for it.

The numbers come from the experiment builds on logos-chat `a74ad35` and sds-rs `648283e`, which
chat-sim pins, with 10 seeds per cell. This change ports the same two behaviours onto current
`main`. It has not been re-run in chat-sim, which needs to move to this revision first.

## Limits

- Agreement still rests on the timer. A commit that cannot be fetched within the timeout still
  forks the group. This change makes that rare. It does not make it impossible.
- An SDS acknowledgement means "received", not "applied". A message can be acknowledged while it
  still waits in the causal buffer.
- More traffic: about 1.4–1.5× the bytes of fetch on sync alone, about 2× before.
