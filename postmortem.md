# Postmortem

## 1. Human Agreement

Two people checked **160 messages** separately.

They both gave the **same answer for all 160 messages (100% agreement)**.

They did not see the expected answers or each other’s answers, so the checking was fair and independent.

## 2. Bugs Fixed

* **Judge v1 → v3:** The old judge did not know the agent could access customer accounts, so it marked some correct replies as bad. We added this information, and the judge became much more accurate.
* **generate_data.py:** Some replies answered the wrong topic. We fixed it so each reply matches its topic.
* **runner.py:** The program sometimes stopped because it misunderstood Groq's limit message. We fixed this.

## 3. Judge vs Human

The judge was compared with human answers on **110 messages**.

* **Accuracy: 99.1%**
* **Cohen's Kappa: 0.98**
* This means the judge and humans almost always agreed.

There was only **one disagreement (ID 31)**.

The human said **good**, but the judge said **bad** because the customer asked for their options, while the reply directly performed an action instead of giving options.

Other similar examples were judged correctly, so this seems to be a small edge-case difference rather than a major problem.

## 4. Still Open

The bias test only used **10 examples** and used generated labels instead of human labels.

It should be tested again with **more examples and human labels**.

## 5. Main Lesson

We should plan the **Groq daily limit** carefully because it can run out quickly.

The limit works as a **rolling 24-hour window**, so it does not simply reset at midnight.

Using `--resume` helped us continue the evaluation without starting again.

## 6. Security Note

The team's **Groq API key was shared in an AI chat** during debugging.

The key should be **deleted and replaced with a new one** before sharing or submitting the project.
