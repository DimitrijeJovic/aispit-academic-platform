# Testing

Source: the bachelor's thesis (sections 3.5, 3.6, 6.1, and 7). The test code is part of the unpublished source, so nothing here can be re-run from this repository.

## Scope

| Kind | Count | How |
|---|---:|---|
| Server-side checks | 547, in 26 groups | Flask test client, in process |
| Checks in a real browser | 27 | Run in a browser against the running application |

The checks were run after every change. The thesis notes that the server-side count varies slightly with the database contents; an earlier stage of the suite is described with 555 to 566 checks.

## Negative verification

A check was accepted only after it had been shown to fail when the rule it covers was deliberately broken. Applying this rule found five checks that passed without testing anything. One example: a check that compared only the HTTP status code passed with the protection switched off, because the accepted and the rejected request both returned a redirect. The fix was to check the state of the database instead.

## Defects the server-side checks could not see

1. **Self-disabling warning dialog** (reported from use). The dialog shown after a student left the exam was placed inside the part of the page that is blurred and disabled on leaving, so it inherited both and could not be clicked or reached by keyboard. At the same time, one departure fired several browser events that were not merged, so a reload counted twice and the three-departure limit was reached at once. Server-side checks saw a normal response and correct page content. Fix: the dialog was moved out of the blurred area, the blur was applied to one element only, all departure events were merged into one input with a time window, and checks running in a real browser were introduced.
2. **Uploaded files reachable without login** (found after development had finished). Files were stored under the static directory. Fix: the storage directory was moved and a regression check added.

## Not covered

- Load testing with many simultaneous users.
- Measured code coverage.
