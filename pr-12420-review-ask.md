@PattaraS, could you take a look at this one? It's a follow-up to #12097, which you merged: the stderr tail that PR added is bounded in lines, and an installer drawing a progress bar emits carriage returns rather than newlines, so a whole install's output can arrive as one line and be held in full in the host daemon. This bounds it in bytes instead (+101/−25), keeping the cause lines #12097 surfaces.

All required checks pass and it's mergeable against current `main`.
