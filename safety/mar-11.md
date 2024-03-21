
# Introduction and Motivations

Machine learning is often under the naive assumption that the data used to train a model is well-behaved. However, a class of attacks known as poisoning attacks have been developed in the last decade or so that involves adding carefully maliciously perturbed points to the training set to make models misbehave. Even the addition of one or a few points can impact the behavior of a model, leading it to exhibit poor performance or misclassify certain points to the adversary's benefit.

Consider the case when a company selling antivirus software relies on the use of ML to train malware classifiers. The data used to train such classifiers is often publicly sourced and given the large amounts of data that training such classifiers may require, it may not be humanly possible to audit every data point in the training set. Adversaries can then sneak in carefully perturbed points, given some knowledge about the model's architecture, training data distribution, parameters, etc. (white-box setting) or even just query access to it and some limited information about the features of the training data and the learning algorithm (black-box setting), and make the trained malware classifier classify some malware as benign software. Issues related to data poisoning have been investigated in the contexts of self-driving cars, medical AI, etc.

Adversarial attacks manipulate machine learning models by exploiting vulnerabilities in their decision-making processes. They can be split into two categories: causative adversarial attacks and exploratory adversarial attacks. Poisoning attacks fall under the causative adversarial attack category because it involves either injecting new malicious points or manipulating existing ones in the training data to degrade the model's performance generally or for specific targets. Subsequently, poisoning attacks can be categorized into either poisoning availability attacks or poisoning integrity attacks. Poisoning availability attacks aim to disrupt the availability of the model's prediction results indiscriminately, often leading to denial-of-service (DoS) scenarios. Conversely, poisoning integrity attacks target specific mispredictions at test time, therefore undermining the integrity of the model's output for malicious purposes while also in most cases keeping their attack undetected.
    
In the context of online training, where machine learning models are continuously updated based on incoming data, poisoning attacks can be particularly potent. Depending on the training system, control over a small number of devices that submit fake information adversarially poison the dataset.

For ML and ML security/safety practitioners, it thus becomes key to ensure that their models are *adversarially robust*. To this end, studying possible vulnerabilities (here to data poisoning) and attacks is an important area of security research. ML security research over the last few years has resulted in attacks that utilize human-indistinguishable adversarial points. This document is meant to introduce readers to the rationale behind poisoning attacks, explain their workings, and how they might be prevented.


# Methods
In this section, we discuss the methodology of some poisoning attacks and some defenses against them, in that order. For the attacks, we describe the respective threat models and the procedure involved in mounting them. Results for each attack and defense are deferred to the next section (Key Findings).
## Attacks

### Attack 1. Poisoning Attack against SVMs [26]
Support Vector Machines, or SVMs, are prime targets for poisoning attacks. In certain cases, even moving one point (a support vector) can entirely alter the behavior of the SVM. In this attack, an adversary can "construct malicious data" based on predictions of how an SVM's decision function will change in reaction to malicious input.
#### Threat Model
This is a white-box attack where it is assumed that the adversary has knowledge about the training data, the validation data, and the algorithm used to train the model (the incremental SVM training algorithm).
#### Procedure
1. Given the training data $`\mathcal{D}_\text{tr}`$ and the validation data $`\mathcal{D}_\text{val}`$, first train an SVM using $`\mathcal{D}_\text{tr}`$.
2. Then pick any member of the attacked class (preferably sufficiently deep within its margin) and flip its label. Denote this point and its flipped label as $`x_c^(0), y_c`$. This is the initial attack point.
3. Using the previously trained SVM, update it on $`\mathcal{D}_\text{tr}\cup x_c^{(k)}`$ for timestep $k$ using the incremental SVM training method.
4. Perform gradient ascent using the hinge loss incurred on $`\mathcal{D}_\text{val}`$ by this updated SVM and finding the direction in which it is maximized. Let $`u`$ be the corresponding unit directional vector.
5. Update $`x_c^{(k+1)}\gets x_c^{(k)} + tu`$ where $`t`$ is a small step size.
6. Iterate until termination (when the increase in loss is below a particular threshold).

This produces an attack point that maximizes the (validation) loss for an SVM and can thus force it to misclassify. As we shall observe in the following section, this attack performs quite effectively. In addition, this work provides some salient points that lend to the attack's potency against SVMs.

- The "gradient ascent" attack strategy against SVMs can be kernelized, allowing for attacks to be constructed in the input space.
    - The attack method can be kernelized because it only depends on "the gradients of the dot products in the input space".
    - This is a big advantage for the attacker when compared to a method which can't be kernelized; an attacker using a non-kernelizable method "has no practical means to access the feature space".
 
In other words, the authors incorporate arguments on how to account for kernelization (for the linear, polynomial, and RBF kernels) and include it within the attack's considerations. 

The authors note that their findings emphasize that "resistance against adversarial data" should be considered in the design of learning algorithms.

### Poison Attack on Linear Regression Models [27]:
Up until the publication of this paper, poisoning attacks were mounted mostly on classifiers. The authors take inspiration from works like [26] and extend the attack methodology to work on classifiers.
#### Optimization-Based Attack:
This attack optimizes the variables selected to make the largest impact on that specific model and dataset by optimizing the difference in predictions from the original unpoisoned model as opposed to the loss. The "label flipping" here is done by assuming that the labels are in $`[0,1]`$ and by taking an existing point, cloning it, and taking its label $`y`$ and replacing it with $`1-y`$ or optionally further rounding this inverted value to 0 or 1. The optimization can proceed using different choices of loss optimization arguments, datasets over which the attack loss is maximized, and the label flipping strategy. Using such an approach yields different optimization attacks. The authors show that for each form of linear regression and dataset that they consider, the hitherto best attack (compared to the baselines) comes from making optimal attack choices. The adversary can thus take an existing point, clone it, "flip" its label to get a poisoning point, and add enough poisoning points to poison the model.
#### Statistical-Based Attack:
This attack is a fast statistical attack that produces poisoned points similar to that of the training data using a multivariate normal distribution. Then, knowing that the most effective poisoning points are near corners, it rounds the feature values of the generated points towards the corners. 


### Clean-label Poison Attack on Neural Networks[29]:
Clean-label attacks are poisoning attacks on neural nets that are targeted, meaning they aim to control the behavior of a classifier on one specific test instance. This type of attack does not require control over the labeling function. This attack produced poisoned points that appear to be correctly labeled by the expert observer but can control the behavior of the classifier. This makes the attack difficult to detect and opens the door for attackers to succeed without any inside access to the data collection and labeling process.

#### Threat Model
The problem of evasion attacks such as Poisoning Attacks against SVMs is that they do not map to certain realistic scenarios in which the attacker cannot control test time data. Unlike Poisoning Attack against SVMs, in the Clean-label attack, the attacker has no knowledge of the training data. However, the attacker has knowledge of the model and its parameters, which is a reasonable assumption since many classic networks pre-trained on standard datasets. The attacker's goal is to cause the retrained network to misclassify a special test instance from one class as another class of her choice after the network has been retrained on the augmented data set that includes poison instances.

#### Procedure
1. The attacker first chooses a target instance from the test set.
2. The attacker samples a base instance from the base class and makes imperceptible changes to it to craft a poison instance.
    - Crafting poison data via feature collisions with equation $\mathbf{p}=\underset{\mathbf{x}}{\mathrm{argmin}}|f(\mathbf{x})-f(\mathbf{t})|_2^2+\beta|\mathbf{x}-\mathbf{b}|_2^2$
    - The rightmost term causes the poison instance p to appear like a base class instance to a human labeler. Meanwhile, the first term causes the poison instance to move toward the target instance in feature space and get embedded in the target class distribution.
3. Inject the poison into the training data with the intent of fooling the model into labeling the target instance with the base label at test time.
4. Train the model on the poisoned dataset.
5. If the model mistakes the target instance as being in the base class, the poisoning attack is considered successful.

Poisoning attacks on transfer learning are relatively effective. In this case, a "one-shot kill" attack is possible, which means by adding just one poison instance to the training set, we cause misclassification of the target with 100% success rate.
However, Poisoning attacks on end-to-end training become more difficult. Single poison instance attack does not work anymore. In this case, special techniques such as optimization, diversity of poison instances and watermarking are needed.




## Defenses
### TRIM [27]

The [Manipulating Machine Learning](https://arxiv.org/abs/1804.00308) paper discusses the defense against poisoning attacks, and particularly for linear regression models, the authors introduce TRIM. The basics of TRIM compared to traditional approaches in robust statistics which is to identify and discard outliers, TRIM adopts a strategy that iteratively refines the training dataset. TRIM operates by selecting subsets of the data that contribute to a lower residual error in each iteration. This iterative selection process allows TRIM to effectively isolate and remove points that are likely to be poisoning attempts, thus safeguarding the integrity of the regression model. As mentioned by the authors, unlike traditional methods that might only offer resilience against random noise or non-adversarial outliers, TRIM is specifically designed to counteract the strategic manipulations characteristic of poisoning attacks. This is achieved by using a trimmed loss function, which recalibrates the influence of each data point based on its contribution to the overall error, thereby ensuring that the model remains robust even when faced with data points specifically designed to undermine its performance.

TRIM operates on the principle of iterative estimation, where at each step, it aims to minimize the impact of potentially poisoned points on the model's performance. The algorithm can be conceptualized as follows:

- Initialization: Begin with the initial dataset and model parameters.
- Residual Computation: Calculate residuals for all points in the dataset.
- Exclusion: Exclude a percentage of points with the highest residuals.
- Parameter Re-estimation: Update the model parameters using the trimmed dataset.
- Iteration: Repeat steps 2-4 until convergence.

The optimization goal of TRIM can be mathematically formulated as: $\min_{\theta} L(D \setminus P, \theta)$, where $\theta$ represents the model parameters, $L$ denotes the loss function, $D$ is the original dataset, and $P$ is the set of points excluded from $D$ based on their residuals. The goal here is to minimize the loss function $L$ over the dataset $D$, while excluding a set $P$ of poisoned points.

In practical terms, this means that even if an attacker manages to inject a certain percentage of poisoned data into the training set, the TRIM algorithm is designed to iteratively "trim" these points out of the calculation for model updating, thereby preserving the integrity of the model's learning process. Furthermore, the paper demonstrates through extensive experiments that TRIM significantly outperforms traditional robust statistics methods, such as Huber regression and RANSAC, in defending against poisoning attacks. TRIM achieved much lower MSEs than existing methods, improving upon Huber by a factor of 131.8, RANSAC by a factor of 17.5, and RONI by a factor of 20.28 on one of the datasets. On the healthcare dataset, TRIM even achieved lower MSEs than those of unpoisoned models, with a reduction of 3.47% for LASSO regression at an attack strength of α = 8%. The reported results show a remarkable improvement in model accuracy and resilience, highlighting TRIM's capability to maintain high performance even in the presence of adversarial data manipulations.

Moreover, the authors demonstrated the effectiveness of TRIM through its ability to outperform existing defenses significantly, not only in terms of resilience to attack but also in computational efficiency. TRIM's running time exceeds among the evaluated defenses, averaging at 0.02 seconds for the house price dataset. The trimmed loss function of TRIM's methodology isolates poisoned data points and does so with a notable speed, making it a good solution for environments where machine learning models must be updated regularly with new data. This dual advantage of robustness and efficiency makes TRIM succeed in the ongoing effort to secure machine learning models against the evolving threat of poisoning attacks.

### Certified Defenses [28]
The defender constructs a feasible set $\mathcal{F} \subseteq \mathcal{X} \times \mathcal{Y}$ and trains only on points in $\mathcal{F}$ :
$$\hat{\theta} = \text{argmin}_{\theta \in \Theta} L(\theta; (\mathcal{D}\_{\mathrm{c}} \cup \mathcal{D}\_{\mathrm{p}}) \cap \mathcal{F}), L(\theta; S) = \sum\_{(x,y) \in S} \ell(\theta;x,y)$$
, where $\mathcal{D}\_{\mathrm{c}}$ denotes the clean data, $\mathcal{D}\_{\mathrm{p}}$ denotes the poisoned data. The defender constructs a feasible set $\mathcal{F} \subseteq X \times Y$ and trains only on points in $\mathcal{F}$.

Given such a defense $\mathcal{F}$, this method considers the upper bound the worst possible test loss over any attacker (choice of $\mathcal{D}\_{\mathrm{p}}$). Such a bound would certify that the defender incurs at most some loss no matter what the attacker does.

There are two classes of defenses.
- Fixed defenses, where F does not depend on $\mathcal{D}\_{\mathrm{p}}$. While such defenders are not implementable in practice, they provide bounds: if even an oracle can be attacked, then we should be worried.
- Data-dependent defenses, where F depends on $\mathcal{D}\_{\mathrm{c}} \cup \mathcal{D}\_{\mathrm{p}}$. These defenders are implementable in practice but they open up a new line of attack wherein the attacker chooses the poisoned data $\mathcal{D}\_{\mathrm{p}}$ to change the feasible set $\mathcal{F}$.

# Key Findings
The [Poisoning Attacks against SVMs](https://arxiv.org/abs/1206.6389) paper [26] presents a novel approach to poisoning attacks against Support Vector Machines (SVMs) 
- Demonstrating that specially crafted training data can significantly increase SVM's test error. 
- Achieved through a gradient ascent strategy that exploits the properties of SVM's optimal solution, even for non-linear kernels.
- Can reliably identify good local maxima on the non-convex validation error surface
- Demonstrate the efficacy of the attack on a synthetic dataset and MNIST. The below figure displays attack efficacy against MNIST when training an SVM to distinguish between 7 and 1, 9 and 8, and 4 and 0. With an increasing number of iterations, the attack produces attack points that effectively increase errors on the validation and the test sets. 

  <img width="832" alt="image" src="https://github.com/wenqian-ye/fall-24/assets/42236940/4fc29f63-bc28-4199-8ea8-3e1eb68b7449">


The [Manipulating Machine Learning](https://arxiv.org/abs/1804.00308) paper [27] proposes optimized poisoning attacks on linear regression models and a novel way to defend against them.
- Statistical-based attacks that are derived from insights gained from optimized research conclusions can act as low-maintenance way to manipulate the training data effectively
- Neat mathematical tricks can be applied to these machine learning problems such as replacing the inner learning problem of the bilevel optimization problem with KKT equilibrium conditions allow us to efficiently approximate our desired learning
- As stated earlier, the best attacks for each scenario (combination of linear regression method and dataset) (OptP) come from the optimization-based approach mounted using the best choices among label flipping strategies, the argument using which loss gradients are calculated, and the dataset over which this loss is maximized. In addition, the poisoning rate plays a role too. The table below shows the best choice of attacks for certain linear regression strategies, and how OptP outperforms baselines.
  
  <img width="643" alt="image" src="https://github.com/wenqian-ye/fall-24/assets/42236940/12e11d63-0826-4a16-8932-458d5b56beb2">
  <img width="777" alt="image" src="https://github.com/wenqian-ye/fall-24/assets/42236940/8982b139-5554-4be1-977d-66aa98e8d95d">

- TRIM utilizes a trimmed loss function to iteratively estimate regression parameters, effectively isolating poisoned points and performing better than other robust regression algorithms designed for adversarial settings
<p align="center">
  <img width="500" alt="image" src="https://github.com/wenqian-ye/fall-24/assets/32115593/256a9235-ff47-47ae-adb1-b93014c8286a">
</p>

The [Certified Defenses for Data Poisoning Attacks](https://arxiv.org/abs/1706.03691) paper discusses how certain defenses against data poisoning can be tested against "the entire space of attacks".
- Specifically, the defense must be one that removes "outliers residing outside a feasible set," and minimizes "a margin-based loss on the remaining data"
- The authors' approach can find the upper bound of the efficacy of any data poisoning attack on such a classifier
- The authors analyze the case where a model is poisoned and where both the model and its poisoning detector are given poisoned data
![Screenshot 2024-03-07 at 8 13 12 PM](https://github.com/wenqian-ye/fall-24/assets/32115593/6c11c060-7450-404f-8755-662197bf94dc)

The [Poison frogs!](https://arxiv.org/abs/1804.00792) paper discusses poisoning on neural networks that manipulate the behavior of machine learning models at test time without degrading overall performance

- Clean-labels are poisoned points that don’t require the attacker  to have any control over the labeling of training data. Only having knowledge of the model and its parameters allows us to create points that are undetected as poison and yet still able to influence the classifier.
- There is an optimized method in making a single poisoned image influence the model, especially for transfer learning scenarios. The addition of a low-opacity watermark of the target instance increases effectiveness for multiple poisoned instances
- Transfer Learning reacts to poison points by rotating the decision boundary to encompass the target, while end-to-end training reacts by pulling the target into the feature space with the decision remaining stationary.
- The below figure displays angular deviation of the feature space decision boundary when trained with clean dataset + poison instance(s) versus when trained with clean dataset alone. In transfer learning (blue), there is a significant rotation (average of 23 degrees) in the feature space decision boundary. In contrast, in end-to-end training (red) where we inject 50 poison instances, the decision boundary’s rotation is negligible. As we can see in the experiment, it is effective on transfer learning but not so effective on end-to-end training. For end-to-end training, all three special techniques are needed: optimization, diversity of poison instances and watermarking. 
![experiment_29](https://github.com/wenqian-ye/fall-24/assets/84001887/dfe55b31-6204-4b65-9b04-e9141e02b338)



# Critical Analysis
The [Poisoning Attacks against SVMs](https://arxiv.org/abs/1206.6389) paper describes a kernelizable method of attacks against SVM models. The authors note that this work "breaks new ground" in the research of data-driven attacks against such models because their attack strategy is kernelizable. Their findings and conclusions are intuitive and easy to follow from the reader's point of view. Their attack strategy description is accompanied by a clear example: an attack on a digit classifier. They clearly show how the attacker is able to use their strategy to modify training input data to produce classification errors. These attack modifications are understandable to the user as well: they clearly show how the attacker modifies the data so that the "7 straightens to resemble a 1, the lower segment of the 9 becomes more round thus mimicking an 8," and so on. Overall, the authors of this paper adequately communicate the technical aspects of their attack strategy in a way that even non-technical readers can easily understand, and the conclusions that follow are well-supported by the evidence they gathered in their attack.

However, the results in this paper are rudimentary as compared to those in recent ML papers. They provide results on synthetic data and subsets of MNIST (by taking two digits and training a binary classifier SVM to distinguish between them). While they illustrated a novel concept well and in depth back then, a more thorough investigation may have been desirable. In addition, the attack on SVM is entirely white box, and involves strong assumptions about the knowledge the adversary has that may be too strong in practice: in real life, adversaries may not have access to information such as the training/validation data.

The [Poison frogs!](https://arxiv.org/abs/1804.00792) paper discusses the potential of clean-label poisoning attacks that challenge the conventional security paradigms within machine learning frameworks. The ability of these attacks to manipulate target model behavior without altering the apparent integrity of the training data underscores a significant vulnerability in current defensive mechanisms. Furthermore, the paper’s exploration into watermarking strategies to enhance the potency of poisoning attacks provides an illustration of the race between attack methodologies and defensive mechanisms in the domain of machine learning security. The authors mention that poisoned training instances can be made more reliable through the application of a watermark that subtly alters the dataset. This challenges the ML community to reconsider the robustness of current model training methodologies against adversarially crafted inputs that do not necessarily conform to expected patterns of attack. The implications of this work suggest that the security of machine learning models cannot solely rely on filtering out malicious inputs but must also adaptively learn to identify and mitigate subtler forms of data manipulation aimed at misleading the learning process.


# References
[26]. Poisoning attacks against support vector machines. Biggio et al. 2012

[27]. Manipulating machine learning: Poisoning attacks and countermeasures for regression learning. Jagielski et al. 2018

[28]. Certified defenses for data poisoning attacks. Steinhardt et al. 2017

[29]. Poison frogs! Targeted clean-label poisoning attacks on neural networks. Shafahi et al. 2018

    
