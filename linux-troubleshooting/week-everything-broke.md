# The Week Everything Broke: Post-Incident Analysis Cheatsheet
# Linux Troubleshooting Series — Episode 18
# youtube.com/@SysTelligence

---

## 1. The Four Outages (Verified Facts)

*   **Microsoft 365 (Nov 12):** A latent software defect from a July gateway update caused lock contention. Combined with a Teams failover and 14% capacity loss, over 80% of gateway servers failed.
*   **Workday (Nov 13):** A Disaster Recovery (DR) test unexpectedly connected to live production databases, causing data inconsistency and forcing maintenance mode.
*   **FAA (Nov 13):** A contractor replacing copper with fiber in Arkansas caused a router to flood the network, grounding all flights for over 4 hours.
*   **Chase/Banking (Nov 16):** A software bug in an on-premise hardware system caused transaction delays, rippling to other banks and Zelle.

## 2. The Common Thread (Change Management)
*   **Microsoft:** Hidden defect + load testing mismatch (tested on weaker hardware).
*   **Workday:** Isolation failure (DR network touched prod).
*   **FAA:** Network change failure (flooding during cutover).
*   **Chase:** Software bug in critical hardware.
*   **None of these were cyber attacks.**

## 3. Prevention: The Change Validation Checklist

### Before the Change
```bash
# Validate syntax before applying
nginx -t
sudo nft -c -f ruleset.nft
sudo visudo -c
sudo sshd -t
ansible-playbook --check --diff

During the Change
Canary First: Deploy to one node. Run health checks. Then widen the rollout.

Rollback Ready: Keep the previous known-good config in your back pocket.

Realistic Testing: Test on production-like hardware (Microsoft's test failed because it used fewer CPU cores).

Isolating DR Tests (Workday's Failure)
Never run DR tests on the production network.

Use isolated VLANs and separate credentials.

Prove the isolation before the test starts.

Know Your Dependencies
List every external service your app relies on (Email, HR, Payment, DNS).

Monitor them from the outside.

Plan manual fallbacks for when they fail.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence


