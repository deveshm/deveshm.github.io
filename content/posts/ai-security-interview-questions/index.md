---
title: "AI security interview questions from Stephen Sims"
date: 2026-01-29
description: "Answering AI security interview questions from Stephen Sims"
summary: "Answering AI security interview questions from Stephen Sims"
tags: ["AI", "interview"]
---

Stephen Sims posted some amazing AI security interview questions on his {{< icon "x-twitter" >}} page: https://x.com/Steph3nSims/status/1958600605511164387

I would love to try and answer them, and help improve my understanding in this space at the same time (I am learning some of these answers as I go).

### 1) What are the main differences between securing traditional software systems and securing machine learning models?

- **Attack surface is broader**, including the software components but also the data and the model itself e.g. data poisoning attacks and adversarial examples, model inversion / extraction attacks and data-point inference attacks
- The **logic is often a "black box"** and decision-making process isn't as transparent. Explainability is a hard problem for Gen AI applications.
- Security is required throughout the whole ML lifecycle e.g. data security, model integrity, input/output validation, and ongoing monitoring
- **Mitigations aren't guaranteed** e.g. training a better model is costly and may still be susceptible to attack

### 2) Define the attack surface of a ML model.

- The **data pipeline** (including training data, inference data / live input)
- The **model itself** (model extraction / theft, model inversion) and confidentiality of the training data (membership inference)
- ML **system infrastructure** and environment (code vulnerabilities and supply chain attacks on the training infrastructure, or API / container / OS vulnerabilities on the deployment infrastructure, or ACL control weaknesses)

### 3) How would you generate adversarial examples against a computer vision model? What defenses exist, and what are their limitations?

A great book I recommend for this question is [Not with a Bug, But with a Sticker](https://www.oreilly.com/library/view/not-with-a/9781119883982/).

- Common white-box attack methods include:
   - **Fast Gradient Sign Method (FGSM)**: A simple and fast one-step attack. It calculates the gradient of the loss function with respect to the input image (see [tensorflow.org article](https://www.tensorflow.org/tutorials/generative/adversarial_fgsm) on it)
   - **Projected Gradient Descent (PGD)**: An iterative version of FGSM. It repeatedly applies small updates to the input image in the direction of the gradient and projects the result back into the allowed perturbation range
   - **Carlini & Wagner (C&W) Attack**: This is a more complex, optimization-based attack that finds the minimal perturbation required to fool the model. See [this APXML page](https://apxml.com/courses/adversarial-machine-learning/chapter-2-advanced-evasion-attacks/optimization-based-attacks-cw) for more details.
- Black-box attacks:
   - **Transferability**: A key insight is that adversarial examples crafted for one model often "transfer" and successfully fool another model, even with a different architecture or training data. An attacker can train a "surrogate" model to mimic the target model's behaviour and then generate adversarial examples using white-box methods on the surrogate model. These examples are then used to attack the real black-box model.
   - **Query-Based Attacks**: These attacks directly interact with the target model. They iteratively perturb the input and **use the model's output (e.g. predicted class, confidence scores) to guide the search for an adversarial example**. Since gradient information is unavailable, these methods rely on approximation techniques.
- Defenses and Their Limitations
   - **Adversarial Training**: augment the training data with adversarial examples and train the model on this expanded dataset
      - Limitations: high computational cost (e.g. examples generated using FGSM or PGD), and specificity (examples may be too specific)
   - **Input Transformation and Pre-processing**: "sanitize" the input image before it is fed to the model, removing the adversarial perturbation e.g. using compression or feature squeezing
      - Limitations: degradation of quality of clean images, and attackers can adapt their example creation pipelines to also use these transformations
   - **Gradient masking**: obfuscate or "mask" the gradients of the model, making it difficult for an attacker to use them to generate an effective adversarial example
      - Limitations: Security Through Obscurity / does not work against transferable attacks
   - Detection-Based Defenses: **A separate model or statistical method** is trained to distinguish between clean and adversarial examples based on their statistical properties
      - Limitations: computational overhead, attacker finding a new adversarial example that fools both models

### 4) Why is gradient obfuscation a weak defense against adversarial attacks?

It creates an illusion of security by breaking the common gradient-based attacks (like FGSM or PGD) that are used for evaluation. A truly robust model has a smooth, well-behaved loss landscape that is difficult for an attacker to exploit. Gradient obfuscation does not alter this fundamental landscape; it simply hides or **distorts the gradient**, which is the tool used to navigate that landscape. The underlying vulnerability of the model to subtle perturbations remains.

### 5) What are some realistic data poisoning threats in enterprise AI pipelines/workflows?

- **Financial Services (Fraud Detection)**: An attacker injects a stream of fraudulent transactions and labels them as legitimate. Over time, the fraud detection model learns to classify these transactions as benign, allowing a larger-scale, undetected fraud scheme to proceed.
- **Spam Filtering**: A disgruntled former employee or an attacker injects a high volume of legitimate emails with "spam" labels into the training data for an enterprise spam filter.
- **Corporate Security (Facial Recognition)**: A malicious actor poisons the training data for an AI-powered facial recognition system used for building access. They inject images of an authorized employee with a small, specific pattern (e.g. a logo on a hat). The model learns to grant access whenever it sees that specific pattern, regardless of whose face it is, creating a backdoor that can be exploited by anyone with the "key".

### 6) How would you go about determining if a model or dataset has been poisoned? 

- Pre-Training Detection (Dataset-Centric):
   - **Data governance and access controls**. Use tools for data lineage to trace a data point back to its source. Any data from an unverified or unknown source should be flagged for manual review.
   - **Outlier and Anomaly Detection** e.g. unsupervised clustering methods like DBSCAN, or statistical measures like z-scores or [Isolation Forests](https://www.datacamp.com/tutorial/isolation-forest) (anomaly detection using binary trees).
   - Label and Feature Integrity Checks: Check for suspicious labels or features that don't align with the rest of the dataset.
- Post-Training Detection (Model-Centric):
   - Performance Monitoring e.g. accuracy, precision, and recall on a known, clean validation set
   - Behavioral Analysis and Canary Tests i.e. this codeflow should never be triggered
   - [Neural Cleanse](https://ieeexplore.ieee.org/document/8835365) and [Backdoor Trigger Inversion](https://ieeexplore.ieee.org/document/9879000): This is a more advanced technique that specifically targets backdoor attacks. It works by reverse-engineering potential triggers.

### 7) How can an attacker perform model inversion or membership inference and what's the potential consequence?

- They exploit the fact that models, especially those that are overfitted, can "memorize" details about their training data, which can then be revealed through clever queries.
- Membership Inference Attack: train a shadow model that mimics the target model's behaviour on a known supervised dataset, then train an attack model which can detect / **infer membership of a data point** on the shadow model. For a given input, the attack model will be trained on the shadow model's output (e.g. probability vectors) and the true label (member or non-member). Finally, the attacker queries the target model with the data point in question and feeds the model's output to the trained attack model.
- Model Inversion Attack: The attacker starts with a random input and a known target class (e.g. a person's name). The attacker **repeatedly queries the model and uses a gradient-based optimization algorithm to iteratively adjust the random input**. The goal is to find an input that maximizes the model's confidence in the target class. Over many iterations, the random input will converge into a reconstructed image or data point that is a good representation of the data used to train the model for that specific class.

{{< youtubeLite id="fyNf4NiJNgU" label="Youtube video on Membership Inference Attacks" >}}

### 8) What mitigations would you apply if an LLM is used for code generation to avoid insecure or undesired outputs?

- Defense in depth
   - Prompt Engineering with security in mind e.g. contextual guardrails and role-based constraints
   - Fine-tuning e.g. create a custom dataset containing pairs of insecure code snippets and their secure, refactored versions.
- Train with secure coding lifecycle practices (use only trusted libraries, scan for banned functions, compile with exploit mitigations, perform static analysis etc) while still having a human in the loop for validation
- Perform static analysis and any code execution or testing should be in a sandbox environment prior to release
- Forbid dangerous constructs (system, eval, dangerous query types)
- Solid dynamic analysis and validation

### 9) When red teaming an AI product or implementation what methodologies have you followed?

- Scoping
- Reconnaissance and Information Gathering
- Scenario-Based Adversarial Testing e.g. Prompt Injection and Jailbreaking, Adversarial Machine Learning e.g. adversarial examples, data poisoning simulations, model inversion and membership inference, model extraction / theft
- Automated Fuzzing and Attack Generation e.g. using [Microsoft's PyRIT](https://github.com/Azure/PyRIT) or [NVIDIA's Garak](https://github.com/NVIDIA/garak).
- Create Actionable Recommendations e.g. improve the system prompt, implement input sanitization, or integrate a separate security filter

### 10) If a company deployed an LLM for customer interactions, what three attack vectors would concern you most, and would it change based on the relevant vertical market?

- **Prompt Injection**: An attacker could force the LLM to reveal its system prompt, act as a proxy for malicious activity, or generate harmful content that could damage the company's reputation.
- **Data Exfiltration**: using the LLM to retrieve and reveal sensitive information from its training data, context, or connected systems
- **Denial Of Service**: incur massive cloud bills for the company or make the service unavailable for legitimate users, leading to a loss of business.

### 11) What are the main trade-offs between model accuracy and privacy-preserving training methods like DP-SGD, federated learning, or homomorphic encryption?

- **Differential Privacy with Stochastic Gradient Descent (DP-SGD)**: Differential Privacy (DP) provides a rigorous, mathematical guarantee that the inclusion or exclusion of a single person's data point will not significantly affect the final model. DP-SGD achieves this by **adding a carefully calibrated amount of random noise to the gradients** during the model's training process.
   - The amount of noise added to the gradients is controlled by a privacy budget (ϵ). A smaller ϵ provides stronger privacy guarantees, but it also **adds more noise, which degrades the model's accuracy**. A higher ϵ weakens the privacy but allows the model to be more accurate.
   - DP-SGD requires careful tuning of hyperparameters, including the privacy budget, [clipping norm](https://www.geeksforgeeks.org/deep-learning/understanding-gradient-clipping/), and learning rate. Incorrect tuning can either compromise privacy or severely harm model performance.
- **Federated Learning (FL)**: allows multiple organizations or devices to collaboratively **train a shared model without exchanging raw data**. Instead, each participant trains a local model on their private data, and **only the model updates (gradients or weights) are sent to a central server to be aggregated into the global model**.
   - If the data across different clients is highly non-IID (non-independent and identically distributed), meaning the data distributions vary significantly, the aggregated model can suffer from "client drift" and perform poorly. This is a primary source of accuracy loss in FL.
   - While FL prevents the sharing of raw data, it is not a complete privacy solution on its own. An attacker on the central server could perform a model inversion attack by analyzing the shared model updates to infer information about individual clients' training data. An attacker could also add malicious updates to poison the global model.
- **Homomorphic Encryption (HE)**: a cryptographic method that **allows computations to be performed directly on encrypted data** without decrypting it first. This means a server can aggregate model updates from clients or even perform the entire training process on encrypted data without ever seeing the raw data or gradients in plaintext.
   - Performing computations on encrypted data is orders of magnitude **slower and more resource-intensive** than on plaintext. This makes HE currently impractical for training large, complex deep learning models in real-time.

---

Live and Learn!

Big thanks for Stephen Sims (@Steph3nSims) for creating these questions, and provoking me to clarify my knowledge on these topics!