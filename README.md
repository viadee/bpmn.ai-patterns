# bpmn.ai: Process Patterns to Orchestrate your AI Services in Business Processes

Ben Wolters, Frank Köhne

The journey through an AI project often only begins with a successful proof of concept. There is still little consensus regarding the orchestration of AI services. Tool support and methodological discussions often culminate with the provision of machine learning pipelines (e.g. through [Kubeflow](https://www.kubeflow.org/)) and web services in the cloud. 
From an architectural perspective, making a single machine learning model usable is straightforward (e.g. via [KFServing](https://github.com/kubeflow/kfserving)) and Cloud providers such as [Azure ML Services](https://azure.microsoft.com/de-de/services/machine-learning/) even hide the service creation and then claim this to be an end-to-end solution to Machine Learning. But, how can you integrate and combine such services into business processes, in a meaningful way?

Companies that already use a workflow engine (such as [Camunda](https://camunda.com)) have a head start on AI use cases. Consider it from this perspective: Introducing AI-based ideas into fully manual processes is much harder than doing the same in a controlled environment with defined data flows. A few integration patterns leverage this controlled environment to orchestrate AI decision making in business processes. 

In our projects, we found the following list of patterns useful. On the one hand, they serve as a means of communication between Data Scientists and Process Owners or Process Automation Specialists. On the other hand, they can be used as a checklist of ideas to consider when you bring AI-based decision making into production.
The patterns can be easily understood as small BPMN processes. They serve different purposes, but most of them help to apply AI decision-making in a responsible and transparent way.

- [bpmn.ai: Process Patterns to Orchestrate your AI Services in Business Processes](#bpmnai-process-patterns-to-orchestrate-your-ai-services-in-business-processes)
  - [Group 1: Getting Started](#group-1-getting-started)
    - [Process data collection - Just Look, Don't Touch!](#process-data-collection---just-look-dont-touch)
    - [Serviceless AI - DMN as Minimal AI Runtime Environment](#serviceless-ai---dmn-as-minimal-ai-runtime-environment)
    - [Digital Common Sense - Anomaly Detection on Process Results](#digital-common-sense---anomaly-detection-on-process-results)
  - [Group 2: Intervenability](#group-2-intervenability)
    - [Controllable Degree of Automation](#controllable-degree-of-automation)
    - [Decision Support - AI-First](#decision-support---ai-first)
    - [Decision Support - Human-First](#decision-support---human-first)
    - [Drift Detection](#drift-detection)
  - [Group 3: Data Protection and Compliance](#group-3-data-protection-and-compliance)
    - [GDPR Consent](#gdpr-consent)
    - [Arguing for a Decision](#arguing-for-a-decision)
  - [Group 4: Multi-Model Patterns](#group-4-multi-model-patterns)
    - [Anomaly Decision Safeguard](#anomaly-decision-safeguard)
    - [Ensemble](#ensemble)
    - [Divide and Conquer - Process Choice](#divide-and-conquer---process-choice)
  - [Summary and Outlook](#summary-and-outlook)

*Convention*: AI components are highlighted in green in the process patterns.

Most of the patterns are not mutually exclusive, feel free to combine them.

## Group 1: Getting Started
### Process data collection - Just Look, Don't Touch!

The first and simplest pattern does not contain an AI component, but instead comprises a business process with a manual activity, whose call is logged with the relevant metadata and process variables included.

![CollectOnly](models/collect-only.png "Collect only")

This pattern already contributes to the implementation of an AI strategy: Since it brings a manual activity under process control, it allows you to derive insights from the process engine logs. Thus, they are highly useful to guide your AI projects in the future.

Relevant questions are:

* How often is this activity performed?
* How large is its share of the runtime of the overall business process?
* How expensive are these activities?
* Are there systematic correlations between input and output of individual activities? May we hope to automate them?

Not only can you use such insights to prioritize automation efforts, but you will also need to rely on them while implementing automated decisions with machine learning approaches: They become part of the *loss function*.


An example: In input management an AI classifies incoming business documents and forwards them to the responsible processes. The machine learning model to accomplish this will learn from the history of manual classifications and try to classify them in the same way. 
Every machine learning needs a target key metric, which has to be optimized. The classic approach for classification problems of this kind is to optimize a kind of _hit rate_ (accuracy) of correct to incorrect classifications. Although this works, it systematically wastes potential savings  - in the end, the goal is incompletely defined. We not only want to approximate past decision making. We want to avoid expensive processes and errors. These goals must become part of the target key metric, because in input management not all misclassifications are equally expensive:


* To wrongly categorize an incoming new customer contract as a termination is the biggest possible mistake.
* Classifying a termination as an incoming new customer contract is incorrect, but has hardly any economic consequences.
* If the economic consequences of mistakes are difficult to estimate, you can at least try to prefer favorable (or favorably correctable) processes.

The classic approach to "maximize hit rate" implies that all errors have the same consequences. This is sufficient for a proof-of-concept, but not for productive, economic use of the AI components. To be able to make these weightings, a process data collection is needed as a basis.

:bulb: Especially external, outsourced tasks deserve this kind of attention.

:warning: Avoid storing personal data in process variables, in order to comply with GDPR goals.

### Serviceless AI - DMN as Minimal AI Runtime Environment

Deep learning is modern and inspires with its possibilities, but it is the current ultimate ratio of machine learning: costly, data-hungry, difficult to understand and non-deterministic. This implies an increase in complexity in IT operations: You will need to maintain several fast-moving infrastructure technologies, probably both software and GPU-resources.

However, simplicity, [explainability](https://arxiv.org/pdf/2010.09337.pdf), and determinism are obvious design goals - according to common sense and also explicitly on the part of regulators such as the [Recommendations of the German Data Protection Conference (in german)](https://www.datenschutzkonferenz-online.de/media/en/20191106_positionspapier_kuenstliche_intelligenz.pdf) since 2019. For decision automation purposes, that rely on tabular data, there may be a shortcut route.

![DMN as runtime](models/dmn-as-runtime-environment.png "DMN as runtime")

One way to maximize simplicity would be to use a rule-based or decision-tree based approach instead. Often, their results can be translated into _DMN decision tables_ with little frictional loss. DMN tables are not only traceable but can also be modified as needed. In addition, there is no need for specific operational infrastructure: Modern workflow engines can execute DMN out of the box. This pattern is highly attractive from an architecture point of view and to collect low-hanging fruits in terms of business cases. However, it will not serve as flagship project and will probably not win the enthusiasm of data scientists.

:bulb: In principle, this is also possible with more complex machine learning methods (e.g. through the [Anchors](https://github.com/viadee/javaAnchorExplainer) approach to derive rules from ML models). 

:warning: However, precision will be lost in more complex cases in exchange for transparency and changeability. Also, there is an upper limit on what you can claim to be simple and explainable, be it in neural networks or large DMN tables. 

### Digital Common Sense - Anomaly Detection on Process Results

If the dark processing rate is high, whether with or without AI technologies, a control problem arises: There is a lack of common sense to check the result of the process again, such as a calculated tariff or a contract, and to check it for plausibility. In general, this should not be necessary, but errors always occur and, if possible, the customer should not be the first in the process to notice an error.

To address this issue, an AI procedure of anomaly detection can be used as one of the last process steps and in individual cases automatically trigger an escalation event - a manual check is requested.

![Anomaly Detection last](models/anomaly-detection-last.png "Anomaly Detection last")

The AI does not make its own decision here, but it can stop decisions of others (people and systems) if they look "strange". The introduction is therefore easier to argue, but may offer fewer savings than optimizations. The keynote of anomaly detection is that an AI model learns what constitutes - or even violates - normality in a business process or its results. This can be based on obvious things like tariffs or costs, but it can also include more factors than would be manageable with manual testing. Optionally, aspects of the course of the process could also become part of the anomaly detection, for example to automatically escalate processes that are particularly long-running or (compared to the learned normality) circulated processes that are particularly frequent.

:warning: Attention: Expect false alarms. This pattern assumes that normality has been established in your process. For some business processes, this assumption can be wrong.

* Very new processes with less than a few hundred process instances will not have produced enough data to allow the algorithm an adequate view of what _usually happens_.
* If this amount of data is gathered, the algorithm can point out _unusual_ data points and bring them to attention. This will also regularly happen after software releases, whenever they influence the data flowing through your processes. E.g. if you change your pricing scheme in a software release the first prices generated will likely look _unusual_ to the algorithm, until it is retrained on the new data points and we gathered enough of them, to consider them a part of the new normality.

## Group 2: Intervenability

### Controllable Degree of Automation

The use of process engines is an investment in flexibility - changes can be made in a coordinated manner without the need for complex release processes. In this way, an XOR gateway can be used to control automation levels.

![Controlled Confidence](models/controlled-confidence.png "Controlled Confidence")

Consider an AI component for a classification decision such as _"Does this insurance claim have to be examined by an expert?"_. Besides the decision itself, the AI component indicates how confident it is with its own decision (Confidence). This is usually given in the value range 0.0 to 1.0, where 1.0 corresponds to a 100% certainty, which is hardly achievable. 
Depending on this confidence value, we branch off to manual processing or bypass it as needed:

* Minimum confidence = 100% - this would be equivalent to a test or pilot operation. The AI component operates live on the real data, but will never make a decision autonomously because the 100% threshold is never reached. Even human clerks would have a residual uncertainty, but do not quantify it.
* Minimum confidence = ~93.45% - The AI decides if it can do it safely and smuggles standard cases past the clerk because, for standard cases, high confidence will be possible. The concrete threshold value can be optimized with regard to process and error costs, so that threshold values with several decimal places may be useful.
* Minimum confidence = 90.00% - A manually set value, based on the experience of those responsible for the process. The value is below the above-mentioned optimum. This increases the automation rate, we accept a higher error rate. This could be a useful configuration after a major incident, where the department is simply overwhelmed.
* Minimum confidence = 0.00% - The AI always decides autonomously, even if uncertainties are clear. Often, this is not a reasonable configuration. You will only want to use this configuration in low-stakes situations such as ad targeting.

:warning: Note group 1: process data collection - automated decisions should be saved for later review, especially if they have been overruled by manual decisions.


### Decision Support - AI-First

In this pattern, a machine learning component is always called before a human decision. It then passes on its results (including confidence estimations and ideally a justification for the result) to support the manual decision. This is particularly useful when an additional method of explainable AI (XAI) is used, which can generate so-called local explanations: for example relative features' importance for individual cases.

![Decision Support - AI first](models/decision-support-ai-first.png "Decision Support - AI-first")

In the best case, a human-machine-four-eyes principle is created that improves the overall level of decision quality and consistency across several decision-makers and thus promotes fairness in the sense of equal treatment. Also, decisions can be made more efficiently and thereby more frequently, if that fits your business case (i.e. in Risk Management or Quality Control).

:warning: These kinds of systems call for ethical considerations despite their focus on human decision making. Clerks will have to justify deviations from the "standard". This creates an incentive not to do so, for example in the [Austrian AMS decision model](https://algorithmwatch.org/en/story/austrias-employment-agency-ams-rolls-out-discriminatory-algorithm/). From an ethical point of view, this would degenerate the process to a fully automatic one (including the quality requirements to be met), as long as it is not organizationally ensured that the human decision will be made independently.

:warning: For this to succeed, equal treatment in the learning data set is a mandatory prerequisite. The Austrian AMS can serve as a negative example again, as it only reflects the prejudices of the labour market on unemployed females.


### Decision Support - Human-First

One approach to handle the problems of the "AI-first" decision support pattern is to turn it around: Human decisions are required first, then the new decision (or a set of recent decisions) is reflected upon using machine learning technologies.

![Decision Support - human first](models/decision-support-human-first.png "Decision Support - Human-first")

Suppose you have a number of decision-makers that randomly receive decision-making tasks. An interesting twist for this case is to take the human decision for granted and try to predict the decision-maker. If this fails or just leads to low confidence predictions, all is well - if it works, it is worth reflecting upon how the machine learning was able to do that. Usually, you will shed light on some kind of misunderstanding or bias. 

Here the machine learning component never influences _individual_ decisions but only serves as quality control for human decision making in the business process. This is appropriate for _high stakes_ decisions that are susceptible to bias: Loan approval, fraud detection, [HR decisions](https://blog.viadee.de/ki-im-personalwesen), etc.

:warning: While this pattern helps to protect those affected by the decisions made, it could both be employed as an opportunity for a team to learn and grow or as a means of workforce surveillance depending on corporate culture. 

### Drift Detection

Most AI applications do not learn continuously (and rightly so). This, however, raises the question: _"How often do I need to train my ML model with new data?"_

Drift (or Concept Drift) is a technical term that describes the fact that data becomes stale after some time. ML models derived from data therefore also become stale: The knowledge baked into the model and reality drift apart. The following pattern helps to measure this effect.

Clerks are randomly assigned cases with a certain probability. This non-automation rate is freely selectable, but should generally not be 0%, so that fresh training data is available in the future. An example: A company that has recorded market or customer behavior over a long period of time and used it for ML-based marketing automation, must consider that the data from before the COVID-19 pandemic is no longer meaningful - the model has experienced a (sudden) concept drift.

![Drift Detection](models/drift-detection.png "Drift Detection")

In order to notice these drifts in less obvious cases, manual decisions are sporadically required. They create a reference.
Often only parts of the market behavior change - small things like new car brands make it difficult for a machine learning model to generalize rules from the past into the future. A continuous supply of up-to-date, manual decisions helps here as well: For example: in 5% of all cases, the decision is made both manually and using the AI model. As long as their level of agreement is stable over a reasonable period, all is well.

If you predict numerical values and not classifications, drift detection works similarly. You need to provide ground-truth values and compare them with your prediction model. If the prediction error is repeatedly larger than a reasonable threshold, you have detected a drift. Your machine learning model is in a new situation, that it is yet not prepared to handle.

The example below shows such a setup for a model that predicts the number of COVID-19 infections in the town of Münster (Germany) across 2020 with a simple model.

![Covic time series drift](examples/covid-time-series/anomaly-diagnostics.png "Covic time series drift")

Researchers from KIT came up with this idea and took it one step further. They use [Drift Detection methods to measure the effectiveness and time lags of interventions](https://publikationen.bibliothek.kit.edu/1000126905) for COVID-19 such as masks and lockdowns, i.e. attempts to actively _break_ the time series and exponential prediction models. 

:bulb: The pattern can also be used as a feature toggle for a pilot, i.e. a riskless rollout if the degree of automation adjusts close to 0%. 

:warning: It is important to log for each case whether a manual or an automatic decision has been made, in order to ensure auditability and to prevent the model from following its own unverified decisions in later training rounds.

## Group 3: Data Protection and Compliance

### GDPR Consent

According to GPDR Art. 22 (Lawfulness of automated processing) para. 1, there is a right of opposition. In other words: Consent of individuals may be necessary to process their data for specific purposes. Among other potential reasons for the lawfulness of processing, consent should be the normal case.

![GDPR Consent](models/gdpr-consent.png "GDPR Consent")

The application of machine learning models would certainly be considered to be _use_ (in the sense of the GDPR), and the inclusion of one's data in the training data stock would certainly be _use_ too. If a customer objects to this use, a "plan B" in the business process is needed, for which ML Serving tools often do not feel responsible.

:bulb: This can be implemented at the processor orchestration level of the IT architecture in a similar way to how VIP business transactions are handled in service companies.

### Arguing for a Decision

In addition to the objection, a subsequent dispute of a decision is conceivable - historical decisions are questioned and, if necessary, revised. This dispute results in the obligation to retroactively reconstruct and correct automated decisions. 
Upcoming legislation will likely stress the point that, from a [consumer's point of view,](https://www.uni-speyer.de/fileadmin/Lehrstuehle/Martini/2019_Gutachten_GrundlageneinesKontrollsystemendgueltig.pdf) an automated decision that can not be reconstructed is no better than an arbitray decision, since it is impossible to justify.

![GDPR Contest](models/gpdr-contest.png "GDPR Contest")

The _orchestrated_ machine learning workflow historizes the case data including the decision and the ML model version. This information is necessary to repeat and systematically analyze individual decisions subsequently on the model. If a case is later questioned, an XAI analysis provides the decision paths that led to the result by repeating this and potentially comparable cases. The individual case is justified and can be discussed and revised if necessary. Wrong decision paths and edge cases become transparent.

Beyond the expected data logging by a process engine, there is more to learn: Revised decisions often show especially important characteristics in the data. The data set is corrected and marked as revised and corrected, to understand and minimize errors in the ML model in the future.

:warning: As soon as personal data within the meaning of the GPDR are processed in the ML model, this pattern is mandatory (cf. GDPR Art. 22 Para. 3).

## Group 4: Multi-Model Patterns

### Anomaly Decision Safeguard

Machine learning models make wrong decisions, just like humans do. Sometimes seemingly random effects put an upper bound on precision - customer churn can be reasonably estimated, but never really predicted for all individual customers. Low training data for rare or special cases is often responsible for estimations that turn out to be wrong later. 

By combining two ML procedures we mitigate this risk by identifying rarities - anomalies - and steering them to the person in charge. It is safe to assume that a machine learning model will not perform in these cases anyway, since individual cases can be strange in all kinds of ways. The underlying reasoning of analogy, when referring to similar cases from the past, is likely to break down when there are no reasonably similar cases. This leads to the following pattern:

![Anomaly Detection First](models/anomaly-detection-first.png "Anomaly Detection First")

An _anomaly detection_ evaluates each case first with an anomaly score. Consider an insurance claims process, the first car damage reported for a new electric car model should of course be evaluated manually, as well as, for example, unusually costly repairs on sports cars are.

* Low anomaly scores should be the rule. These routines are sufficiently present in the training data and the Ml model can decide cases with high confidence.  
* Medium anomaly scores are rare. Process owners control a threshold value equivalent to the confidence. If this threshold exceeds, it is an anomaly that requires human attention. In this way, useful training data is generated.
* High anomaly scores - anomalies - are always taken over by manual processing. Here there is little hope for automation in the long run, but the attention is well invested. 

Anomaly Detection addresses the risk of unknown cases "slipping" through automation. Although an ML-based classification should evaluate anomalies with low confidence, anomaly scores and confidence values are different use cases.

Anomalies could identify special cases that may offer opportunities for process improvement. Controlling anomalies generate new learning data in a targeted manner, which will help future models learn to deal with these cases more safely.

:bulb: Machine learning models that are accessible by third parties may also have to protect themselves against targeted attacks that intend to mislead a model Adversarial Machine Learning. For example, MIT researchers managed to [forge images of turtles](https://arxiv.org/pdf/1707.07397.pdf), that popular classifiers recognize as a rifle reliably. If you need to [defend](https://arxiv.org/pdf/2009.03728.pdf) your automated process against this kind of fraud, you can employ the above _Anomaly Detection First_ pattern as well, in order to prevent the classifier step from seeing adversarial cases in the first place.

### Ensemble

Different methods of machine learning also have different strengths and weaknesses. Here are two examples:

* Decision trees, for example, are easy to understand - the overarching problem is easy to master, but complex relationships in the respective domain remain unnoticed.
* Deep learning is difficult to understand but learns relationships in deeper complexity.

The ensemble idea is to combine the strengths of several methods: Several models decide the same case - if there is an agreement, the decision is clear. Disagreement is observed and also monitored with machine learning tools - another model decides which models perform well in which situations. 

![Ensemble](models/ensemble.png "Ensemble")

Alternatively, decisions can also be made democratically, if necessary with human participation in the sense of a four-eyes principle. Often ensembles are also used "packaged", i.e. in such a way that they act as a single model from the calling architectural layer. A disadvantage is that the strengths and weaknesses of the partial models remain invisible to the outside and thus also non-reflectable. Explicit modeling also makes a step-by-step rollout of new models possible: New "fellow citizens" join the democracy and vote, perhaps giving good confidence values and thus receiving a lot of decision weight - or not.

:bulb: Ensembles tend to be harder to fool than individual models, both against random noise and against attackers using adversarial learning techniques. 

:warning: Run and learning times increase, feature importance is more difficult to derive. Decision quality increases at the price of the complexity of ML-based decisions.

### Divide and Conquer - Process Choice

Often it is not just one process that needs to be automated, but several. Often processes can be triggered by business transactions via several input channels. This is where the Divide-and-Conquer principle comes into play with several ML models that solve parts of a problem.

![Process Choice](models/process-choice.png "Process Choice")

A first processing step extracts all entities from incoming documents, e-mails, or chat messages, i.e. all identifiable aspects, as they might be entered into a form in the processing: Customer names, e-mail addresses, invoice numbers, etc. or more abstract assessments such as a sentiment-analysis to identify and quantify affective states and subjective information, especially the mood expressed in a piece of text.

In this way, a stream of rather weakly structured data turns into one with known (and typed) fields. These can be used in a second ML step to classify the most suitable subsequent process that can process the data with non-AI process automation means.

The separation of these two steps makes strategic sense because _Entity Extraction_ is becoming a standard product for which pre-trained models or cloud services can be used. Process classification requires separate learning data from your own processes.

:warning: There may be incidents that do not belong in any process or contain multiple concerns.

:warning: The target metric used for machine learning should not only be optimized for the "hit rate" but should also take individual error costs into account. The false classification of a contract application document as a contract termination can be expensive.

## Summary and Outlook

Machine Learning supports many challenges of modern process automation. The collection of _process key metrics_ alone improves the data situation and provides information about the behavior of individual sub-processes. This should be considered when evaluating business cases for workflow engines. Further, they enable some synergies for orchestrated AI: 

* You can manifest your healthy skepticism about ML decision-making into checks and balances through explainability, drift detection, and anomaly detection. 
* Process owners require confidence thresholds to control the degree of automation. 
* A history of ML results and used models helps to argue for accountability and closes the feedback loop to maintain the quality of automated decisions in the long term.

:soon: There are more patterns in the backlog, which we plan to include in the future. This list of patterns has a permanent open-source home on [Github](https://github.com/viadee/bpmn.ai-patterns). We hope that you find this high-level perspective useful in your own projects. We are happy to discuss them in more detail and we are looking forward to your feedback! 
