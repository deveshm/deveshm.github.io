---
title: "Practical Autonomous Defense and Unprompted AU"
date: 2026-09-22
description: "How we get to autonomous defense, and a summary of my experience at the Unprompted AU Security Conference"
summary: "How we get to autonomous defense, and a summary of my experience at the Unprompted AU Security Conference"
tags: ["defense", "AI"]
---

I want to tackle the big elephant in the room: **how do we actually get to autonomous defense?**

We have seen many examples of autonomous offense / hacking, including the OpenAI-HuggingFace incident, and threat intel reports from [Anthropic](https://www.anthropic.com/threat-intelligence-report-september-2026) and [Google](https://cloud.google.com/blog/topics/threat-intelligence/from-prompting-to-autonomy-the-evolution-of-adversarial-ai), revealing how human-in-the-loop involvement is dramatically reduced in attack chains.

However, defense is traditionally hard to automate due to the high cost of false positives (you may accidentally disrupt critical business operations or lock out legitimate employees if the AI misinterprets normal activity as a threat) and the complexity of business context involved in making security decisions.

As discussed in the latest [Srsly Risky Biz](https://news.risky.biz/tag/seriously-risky-business/) episode, some companies require speedy, automated detection and response mechanisms to thwart threats (think crypto exchanges), while other companies have the luxury of taking their time to respond appropriately to a hack (e.g. the HuggingFace response). However, as models get more powerful and reachable by wider audiences, all businesses will need to make defensive decisions faster as well.

My belief is that autonomous defense will never exist (at least as long as there is 1+ human employees in the company). However, I believe that close-to-autonomous defense can be achieved, and **must be sought after** to tackle autonomous offense.

I've seen recommendations for companies to simply "patch faster", to tackle the "vulnpocalype". However, this approach is naive and ignorant of the challenges businesses currently face. If they could patch faster, they would...however they need to first test the patch doesn't break anything in their environment, and on top of that, they now need to be worried about the risk of supply chain attacks as well. **The patching paradox**: If you push the patch instantly to stay safe from the bug, you risk blindly introducing a catastrophic supply chain attack into your environment. If you hold back the patch to thoroughly vet it for supply chain integrity, you remain wide open to the original bug.

<h2>Introducing the concept of "DIDRE: The Defense in Depth Recommendation Engine"</h2>

What our industry needs is a recommendation engine that understands your business context, and complements your SOAR (Security Orchestration, Automation, and Response) tooling. When you cannot respond to an incident in real-time, you need to make defensive decisions as fast as possible. In the [HuggingFace case](https://huggingface.co/blog/agent-intrusion-technical-timeline#what-we-changed), that involved using GLM5.2 to understand what had happened, and then making decisions on how to improve their security posture (such as fixing the initial-access vulnerabilities, revoking and rotating credentials, locking down the cloud metadata service, deploying stricter admission controls on k8s clusters, and narrowing credential scope). Each decision made was to add a layer to their defense, and make it a lot harder for the next attacker that inevitably tries to hack them again.

So, is it possible to have a recomendation engine that would understand your business context and recommend the best possible defense-in-depth measure for you in that point in time? It has to weigh up many things, including:
1. Monetary cost and human effort to implement
2. What would the business impact be of the change? Will certain users get hindered from doing their job? What current business processes would be impacted?
3. What mitigations currently already exist?

The above questions have, in the past, always been answered by humans that defend networks. LLMs can be used to help them make decisions faster, when they cannot be automated by SOAR.

What are examples of defense-in-depth measures that DIDRE may recommend, you ask? Well I have some examples of security best practices for you:
1. Protect user's accounts with **2FA** (passkeys or FIDO U2F hardware keys) and strong passwords (use a password manager)
2. Protect vulnerable network services by putting them in isolated networks (see **network allowlisting** controls)
3. Require **multi-party human approval** for sensitive mutation actions
4. Monitor for anomalous activity that strays from the baseline of how things usually are e.g. when accessing keys / credentials
5. Modernize your business e.g. **moving off legacy operating systems** to modern ones that support modern exploit mitigations (and ask your vendors to do the same)
6. Red team your public (and internal facing) attack surface to find vulnerabilities and fix them before others find them
7. Keep **offline backups**
8. Use **sandboxes / ephermeral environments** for CI/CD jobs and restrict pipeline credentials (see a more thorough [guide for software supply chain security here](https://www.sentinelone.com/cybersecurity-101/cybersecurity/software-supply-chain-security/))
9. Prepare an **incident response plan**, for when things inevitably go sideways

My hope is that every business has their own DIDRE, that works with your security team and in the business's best interests.

<h2>Unprompted AU</h2>

I also attended [Unprompted AU](https://unprompted.au/) last week, and absolutely loved the conference. I met some amazing people in the industry, working on high impact problems, and listened to many high quality presentations. Here are some of my takeaways and favourite presentations:
1. Most of the talks were related to using **AI for offense**. This may be because it is more fun, or maybe because it's easier to automate that defense. Either way, it got me motivated to think about the defensive side of the equation.
2. My favourite talk was one titled "Security Through Obscurity Is Dead and LLMs Killed It", where the presenter demoed 2 brilliant uses of LLMs: one for emulating a printer in the browser (with qemu-wasm) and then replicating a printer exploit in the browser, and another for bypassing obfuscation implemented in Counterfeit Deterrence Systems (CDS). The demonstrations went so smoothly, and got a huge round of applause from the audience!
3. The two main defense related presentations were Thomas Roccia's talk on detecting threats to the AI ecosystem (e.g. malicious prompts) with [NOVA](https://securitybreak.io/novahunting) and Slack's talk on their **agentic detection and response system**. Slack's system helped them increase their True-Positive detecion rate, tested against a red team exercise, however I am still a bit wary of the amount of False-Positives it may introduce in conjunction with this.
4. With access to source code (either through public commits, or private repos, or reverse engineering binaries), LLMs are extremely powerful, and are amazing aids to help an experienced exploit developer find exploits even faster.
5. People are still finding their way around **closed-model safeguards** e.g. by social engineering the agent to convince it that their security research is for legitimate bug bounties, or using publicly available jailbreaks (see [time.com article](https://time.com/collections/time100-ai-2025/7305870/pliny-the-liberator/) on pliny the liberator).
6. **Reverse Engineering and Simulation** is a LOT easier now with LLMs, so much so that we can replicate full applications and systems (see point 2 above). Another great talk that showed this was one that replicates a mobile app that talks with a consumer portable power station via CANBus. This talk motivated me to think about other app-connected devices around the home which don't come with the best security - in this day and age, we can isolate these internet connected home devices and create replica mobile applications to communicate with them locally, giving us the best of security AND ease-of-use.

---

Live and Learn!
