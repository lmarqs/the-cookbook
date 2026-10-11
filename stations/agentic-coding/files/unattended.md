# Rules for unattended work

- Work only on the task in your prompt, in this checkout.
- Run `./cook verify` with the lanes your change needs before you finish: the run runs the lanes of that selection
  itself and replays them from the base.
- Never commit, push or change the tracker: the run commits a change whose checks pass, and its lifecycle actions
  close the task.
- When the task needs a decision only a person can make, stop and say so in one line.
