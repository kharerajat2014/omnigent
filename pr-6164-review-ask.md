@dbczumar, you've changed `managed_hosts.py` recently (#11223, #9457), so you may be the right reviewer here. The 120-second wait for a managed host to register is a hardcoded constant; a deployment with a slower host image can only raise it by patching the code. This adds `OMNIGENT_MANAGED_HOST_ONLINE_TIMEOUT_S`, following the pattern `OMNIGENT_E2B_MAX_LIFETIME_S` uses, and fixes the rendezvous timeout that a longer override would otherwise outlive.

Rebased onto `main` yesterday; all required checks pass and it's mergeable.
