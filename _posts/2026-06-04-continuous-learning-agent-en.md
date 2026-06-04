---
layout: post
title: "From External Memory to Fast and Slow Learning: How I Understand the Next Step for Continual Learning Agents"

---

[中文版]({{ '/2026/06/04/continuous-learning-agent.html' | relative_url }})

Recently I have been reading several papers on continual learning, agent memory, and model adaptation. They do not study exactly the same object, but when placed together, they form an interesting line of thought: if we want AI agents to do more than complete one-off tasks, and instead gradually adapt to users, accumulate experience, and correct their behavior over long-term use, how should they actually "learn"?

The first paper that drew my attention was XSkill: Continual Learning from Experience and Skills in Multimodal Agents. It points toward a fairly optimistic direction: instead of updating model parameters, an agent can extract experience and skills from past trajectories, store them in an external knowledge base, and later retrieve, adapt, and reuse them in new tasks. Later, I read Useful Memories Become Faulty When Continuously Updated by LLMs, which is almost a necessary bucket of cold water on the previous approach: current LLMs are not reliable memory consolidators. If we continuously ask them to abstract experience into memory, the memory may become worse and worse, even falling below a no-memory baseline.

Between these two, I also read a paper that is not directly about agent memory but was very helpful for my thinking: Learning, Fast and Slow: Towards LLMs That Adapt Continually. This paper mainly discusses fast and slow learning during LLM post-training. Model parameters are slow weights, while prompts and context are fast weights, and both should be optimized during training. It is not studying exactly the same object as XSkill or Faulty Memory, but it gave me an important conceptual frame: learning does not have to happen only in model weights. It can also happen around the model, in context, memory, tool configuration, and the runtime framework.

This blog is my attempt to organize that line of thought: from XSkill's external experience-skill memory, to Faulty Memory's warning about continuous memory updates, to FST's fast and slow learning framework, and finally to one idea of my own: future personal agents may need a fast-slow dual system. The fast system is an agent harness that updates continuously. The slow system may be an on-device small model or a personalized adaptation layer. A cloud-based large model can participate as a teacher, auditor, and complex reasoner.

## 1. Why Agents Need Continual Learning

Many current AI systems have an unnatural quality: they are smart, but they are not very good at "living with" a workflow over time. They can perform well in a single task. They can read documents, write code, call tools, and answer complex questions. But once the task ends, they often do not really absorb the experience. The next time they encounter a similar problem, they still behave as if it were the first time: they judge again, explore again, and make the same mistakes again.

This is very different from how humans learn. When a person works with a particular user, workflow, or task type for a long time, they gradually form habits and methods. For example, a research assistant who repeatedly helps you read papers will learn that you prefer to first look at the problem framing, then the method, then the experiments and limitations. A long-term programming collaborator will learn what project structure, explanation depth, and code style you prefer. These things are not generated temporarily inside a single task. They are accumulated through long-term interaction.

If we truly want a personal agent, it cannot just be a stateless tool caller. It should learn from past experience. It should know the user's preferences, the project's context, which methods worked before, and which mistakes should be avoided. The question is: where should this learning happen?

The most direct answer is to update model parameters. But that is expensive and risky. Parameter updates are difficult to audit, difficult to roll back, and may cause forgetting. Another route is to update the systems outside the model: memory, skills, prompts, tool policies, workflows, and retrievers. This is the external memory route for agents, and it is the focus of papers such as XSkill.

## 2. XSkill: An External Experience-Skill Memory System

XSkill studies multimodal tool agents. A multimodal tool agent is not just a system that can look at images and answer questions. It can also call code tools to process images, and use web search or image search to obtain information. It is not facing pure language reasoning, but rather a setting that requires constant switching among visual observation, tool use, and multi-step decision-making.

The problem with this type of agent is often not that it lacks tools, but that it cannot use tools stably. For example, when an image is upside down, it may fail to rotate it first. When the target is too small, it may fail to crop and enlarge it first. When identifying an unknown object, it may fail to use image search first. After a search fails, it may not know how to recover by trying another strategy. In other words, the capability is there, but the action strategy is unstable and the tool orchestration is not flexible enough.

The core idea of XSkill is that an agent should accumulate two types of external knowledge from past trajectories: experience and skills.

Experience is action-level, local, and context-sensitive knowledge. It usually answers the question: in this specific situation, what should the next step be? For example, when an image is too dark, enhance brightness first; when a target is too small, crop and enlarge it first; when an unknown object cannot be recognized, consider image search. Experience is not a complete workflow. It is a local judgment that helps the agent take fewer wrong turns during execution.

Skills are task-level, structured, reusable workflows. They answer the question: how should a certain type of task usually organize steps and tools? For example, when identifying an unknown object in an image, the agent can first locate the target region, then crop the target, then perform image search, and finally visit webpages and synthesize the answer. A skill is more like an operation manual. It helps the agent with high-level planning and tool orchestration.

This dual-stream design of "experience and skills" is important. Many agent memory methods tend to call everything memory, but XSkill further distinguishes knowledge at different granularities. Experience handles in-the-moment reaction, while skills handle the overall workflow. Experience makes the agent more flexible, and skills make the agent more stable. For a long-running agent, this kind of layering is more reasonable than simply piling up memories.

The second key point in XSkill is visual grounding. In multimodal tasks, many important failure causes are not in the text log, but in the visual state. For example, a text trajectory may only tell us that "search failed", while the real cause may be that the image was upside down, the target was too small, the lighting was too dark, or the agent did not crop the relevant region first. If experience is summarized only from text trajectories, the agent will find it difficult to learn truly useful visual action strategies.

Therefore, when XSkill extracts experience and skills, it combines image observations, tool calls, intermediate results, and task outcomes. It does not merely summarize "which tool was called". It tries to summarize "which visual state triggered which tool action, and why that action was useful". This is what makes it more targeted than ordinary text-based agent memory.

The third important design is retrieval followed by adaptation. XSkill does not simply paste old experience into the prompt unchanged. Instead, it first retrieves relevant experience and skills according to the current task, then rewrites them into concrete suggestions based on the current image and question. For instance, an old experience may simply say, "when the target is too small, crop it first." In the current task, it may be adapted into: "first locate the two small targets in the corner of the image, crop and enlarge them, then use image search or web search for identification." Old knowledge needs this kind of adaptation; otherwise it becomes a rigid template.

XSkill also has an architectural design worth noting: it separates the execution model from the knowledge management model. The execution model is responsible for actually performing the task. The knowledge management model is responsible for summarizing trajectories, extracting experience, generating skills, merging knowledge, retrieving knowledge, and adapting knowledge. The paper also mentions that knowledge management can use a stronger model, while the execution model can remain flexible. This is illuminating because it shows that an agent's learning system does not have to be placed entirely on the same model. One model can act, while another stronger model can reflect and organize knowledge.

So the XSkill paradigm can be summarized as follows: instead of updating model parameters, an agent accumulates procedural knowledge across tasks through an external experience-skill knowledge base. This procedural knowledge is not factual knowledge, but knowledge about "how to do things". It is not ordinary RAG, where the system retrieves what a document says. It retrieves tool-use methods, error recovery strategies, and workflows summarized from past tasks.

This direction is very attractive. It provides a path for continual learning without parameter updates: by changing its external knowledge structure, an agent can become more skilled without changing the model itself.

## 3. Faulty Memory: Continuously Updated Memories May Get Worse

If we only read XSkill, we may become quite optimistic about external memory systems. But Useful Memories Become Faulty When Continuously Updated by LLMs reminds us that external memory is not magic. In particular, memory that is continuously summarized and updated by an LLM itself may develop systematic problems.

The core question of this paper is: if an agent continuously compresses past trajectories into textual memories, and updates this memory base after each new interaction, will the memory really become better and better?

The authors' answer is: not necessarily. Memory usefulness may increase at first, but as consolidation and rewriting continue, it may gradually decline, and may even fall below the no-memory baseline.

The most important conceptual distinction in the paper is between episodic memory and abstract memory. Episodic memory is the original experience itself, including tasks, observations, action steps, tool calls, errors, feedback, and results. It is like an original case file: information-rich and not very compressed, but it preserves evidence. Abstract memory, by contrast, consists of lessons, rules, skills, or experience summarized from multiple original experiences. It is shorter and more reusable, but it requires an LLM to abstract and rewrite.

Many previous agent memory systems pursue abstract memory, because a long-running agent cannot store all trajectories forever. But this paper points out that abstraction itself is lossy. Each summarization may drop details, remove applicability conditions, introduce incorrect rules, or merge experiences that should not be merged.

One strong counterexample in the paper is especially powerful. Without memory, the model could originally solve a set of ARC-AGI problems, and the memory construction process was given the true solutions. In other words, the experiences fed into the memory system were not failed trajectories or noise. They were high-quality, correct experiences. In principle, memories summarized from true solutions should at least not harm the model. But the experiments show that after streaming consolidation, the model failed on problems it could previously solve. This result suggests that the problem is not that the experience itself is useless, but that the consolidation process turned useful experience into faulty memory.

The paper also finds that the same set of experiences can produce memories of different quality depending on the update method. Summarizing after seeing the entire trajectory pool is different from updating in a streaming, batch-by-batch way. Summarizing by task type is different from mixing different tasks together. Streaming updates are more likely to cause memory erosion, and heterogeneous tasks in the same batch are more likely to lead the model to summarize incorrect rules.

This is especially important for real agents. A deployed real-world agent lives in a streaming environment: tasks arrive one after another, user preferences gradually change, and tool results keep appearing. It cannot always summarize all history offline, completely, and by well-organized task families. In other words, real agents are exactly in the setting that this paper considers most likely to produce faulty memory.

The paper also includes an important comparison: directly preserving original trajectories without abstraction can already be a strong baseline. Providing original trajectories to the model as in-context examples can often compete with, or even outperform, abstract memory methods. The reason is intuitive: original trajectories preserve observations, actions, failures, feedback, and concrete context. The model can use in-context learning to benefit from concrete cases. If the abstraction process is unreliable, preserving evidence directly can be more stable.

The authors summarize three mechanisms that create faulty memory.

The first is incorrect grouping. Before abstraction, the model needs to judge which experiences belong to the same category and which can be merged. If it merges tasks with different underlying structures, it will summarize incorrect rules.

The second is overgeneralization. An experience that only applies in a certain context is written as a general rule, and then misleads the agent in nearby but different tasks.

The third is narrow-sample overfitting. If the input stream contains many similar examples, the model may treat accidental features of those examples as general laws, causing memory to generalize poorly to new examples.

The conclusion of this paper is not "agents should not have memory". Rather, it is that an agent's memory system cannot be just a continuously rewritten abstract library. Original episodes should be preserved as first-class evidence, and abstraction should be gated. The system should not automatically summarize after every interaction. It should judge whether there is enough evidence for abstraction, which details must be preserved, which memories need deletion, and which abstractions should be rolled back.

The value of this paper is that it moves agent memory research from "how to remember more" to "how to remember reliably". A memory system must not only write memories; it must also govern them.

## 4. FST: Why the Fast and Slow Learning Framework Still Matters

Learning, Fast and Slow is not addressing exactly the same question as the previous two papers. It mainly discusses the post-training stage of LLMs, rather than deployed agent memory. But it is very important for my thinking because it provides a more general framework for learning: learning should not happen through only one channel.

The core of FST is to divide LLM adaptation into two types: slow weights and fast weights. Slow weights are model parameters. They are updated through methods such as RL, have high cost, have persistent effects, and are suitable for carrying long-term capabilities. Fast weights are prompts, context, or textual scaffolds. They are not real neural network parameters, but they significantly affect model behavior and can be changed quickly.

The paper argues that LLM post-training should not only update parameters; it should also optimize context. Traditional RL post-training usually fixes the prompt and writes all task adaptation into parameters. This may cause the model to drift away from the base model, forget existing capabilities, and become less plastic for learning future tasks. FST instead allows prompt/context and model parameters to co-evolve: it updates context using prompt optimization methods such as GEPA, while updating model parameters with RL under those contexts.

The "continual" in this paper mainly happens during training. That is, it does not claim that the model automatically learns during ordinary deployment. Rather, it says that during a stream of post-training tasks, the model can continue adapting as tasks change. Its value is that it points out a pressure distribution problem: if all learning pressure is placed on parameters, the model may become overly specialized; if context carries part of task adaptation, the parameters do not need to move as far, and the model may retain more plasticity.

The relationship to agent memory is indirect, but it gives me a transferable idea: if prompt/context can serve as the fast learning channel during LLM post-training, then in an agent system, the entire harness can be viewed as the fast learning channel.

Here, harness refers to the runtime system around the model, including memory, skills, prompts, tool configuration, retrievers, workflows, reflection rules, evaluators, user preferences, and task templates. For an agent, fast weights should not just be a prompt. They can be the whole mutable external runtime structure.

So we can migrate the idea of FST from "model training" to "agent learning": model parameters are the slow system, and the agent harness is the fast system. The fast system adapts quickly to the current user and task through memory and skills, while the slow system absorbs long-term stable capabilities or preferences at lower frequency and with more caution.

This migration also made me think about another question: can the idea of reinforcement learning for models be migrated to reinforcement learning for agents? In ordinary LLM RL, the model generates an answer, and the system updates parameters according to reward. In agentic RL, the agent does not just generate an answer. It acts continuously in an environment: observing, calling tools, receiving feedback, acting again, and eventually completing the task. Here, the learning target does not have to be only model parameters. It can also be the agent's tool policy, memory update rules, skill library, task decomposer, and workflows. In other words, agent reinforcement learning can train or improve not only the model itself, but also its harness.

This is why FST, although not an agent memory paper, still belongs in this line of thought. It provides the conceptual foundation for dividing labor between fast and slow systems, and we can extend that framework to agents: fast learning happens in the harness, while slow learning happens in the model or an on-device personalization layer.

## 5. The Core Tension I See: Memory Needs Compression, But Compression Can Fail

When I put these papers together, the core tension I see is this:

Agents must learn from experience, otherwise they will forever remain one-off tool callers. But experience cannot be stored infinitely; it must be compressed, abstracted, and organized. At the same time, current LLMs are not reliable enough, and the abstraction process may turn useful experience into faulty memory.

This is the fundamental difficulty of continuous agent memory.

If we preserve only raw trajectories, the information is most complete and most traceable, but it will grow without bound. Context length is limited, retrieval cost will increase, and privacy and storage will also become problems. A long-term personal agent cannot preserve every detail forever and consider all of it in every task.

If we compress trajectories into abstract memories, the system becomes lighter, easier to transfer, and closer to true experiential learning. But the Faulty Memory paper tells us that abstraction is not free. Incorrect grouping, overgeneralization, lost applicability conditions, and streaming update erosion can all make memories worse.

Therefore, future agent memory should not choose only between "preserve raw trajectories" and "summarize abstract experience". It needs layers.

The first layer should be raw episodic evidence. It does not necessarily need to be preserved permanently in full, but at least before and after a memory is abstracted, the system should be able to trace which original experiences the abstraction came from.

The second layer is gated abstract memory. The system should not automatically summarize every interaction. It should create experience, skills, or workflows only after enough similar evidence exists, the applicability conditions are clear, and the abstraction has been validated.

The third layer may be a slower personalized model. Long-term stable and repeatedly verified user preferences and task patterns perhaps should not remain forever in external memory. They could periodically enter an on-device small model or a local adapter.

In this way, agent learning is not just "the memory base gets bigger". It becomes a process that moves from raw experience to abstract skills, and then to long-term personalized internalization.

## 6. My Proposal: A Fast-Slow Dual System for Personal Agents

Based on these papers, I now have a preliminary proposal: future personal agents may need a fast-slow dual system for continual learning.

The fast system is the agent harness. It includes raw episodic memory, abstract experience, a skill library, user preferences, tool policies, retrievers, workflows, prompts, and reflection rules. Its features are fast updates, low cost, editability, and rollback. If the user states a preference today, the agent should be able to use it tomorrow. If a task fails, the agent should immediately be able to record the mistake and the recovery strategy.

But the fast system cannot expand without limit. The Faulty Memory paper reminds us that memories can become dirty and abstractions can become bad. The fast system is suitable for short-term adaptation and auditable memory, but not for carrying all long-term personalization.

The slow system can be an on-device small model or a local personalized adapter. It is not meant to replace the cloud large model. Instead, it absorbs user patterns that are long-term, stable, and repeatedly verified. For example, a user's preferred explanation style, writing tone, project organization, common toolchain, paper-reading habits, and coding style. After these things are validated enough times, they can migrate from the harness into the on-device model.

The key here is the threshold. The system should not train a model as soon as it sees one memory. It should wait until certain patterns appear repeatedly and have been validated before starting slow learning. The process could look like this:

The user and agent interact over a long period. The fast system first preserves raw episodes, user feedback, task outcomes, and temporary experience. After some time, the system checks which preferences are stable, which workflows repeatedly work, and which experiences have been validated multiple times. Then the cloud large model participates in auditing and organizing, converting those stable patterns into training samples, preference data, counterexamples, and small evaluation sets. Finally, the on-device small model or adapter updates at low frequency, and an evaluation set checks whether the update truly improves the user experience. If the update fails, it is rolled back.

This architecture can avoid two extremes. One extreme is to only pile up memory forever, until the memory becomes larger and dirtier. The other extreme is to update model parameters frequently and write erroneous experience into the model, making the system difficult to audit and roll back. A more reasonable route is: rely on the harness for short-term fast adaptation, and rely on an on-device model for low-frequency long-term internalization.

## 7. Collaboration Between Cloud Large Models and On-Device Small Models

Another key point, in my view, is that we should not treat an on-device small model as a shrunken replacement for a cloud large model. Compressing a powerful general-purpose large model fully onto a device while hoping that its ability will not degrade is technically very difficult. A more reasonable direction is collaboration between large and small models.

The cloud large model is responsible for complex reasoning, high-quality memory review, training data generation, and difficult tasks. The on-device small model is responsible for local personalization, privacy protection, user preference modeling, local retrieval, and context assembly. In this way, the small model does not need to be an all-purpose model. It only needs to become the model that understands this user best.

For example, the on-device small model can first judge whether the current task involves privacy, which local memories can be used, and which information should not be sent to the cloud. It can compress user preferences and local context into a better prompt and send it to the cloud large model. After the cloud large model performs complex reasoning, the on-device small model can adjust the final response according to the user's style. Going further, the cloud large model can also periodically help the on-device small model generate training data and evaluation sets, so that the on-device small model gradually becomes more suitable for this user.

In this way, the architecture of a personal agent may look like this:

The cloud large model provides general intelligence and complex reasoning ability. The on-device small model provides long-term personalization and privacy boundaries. The agent harness provides short-term memory, tool use, and task workflows. Together, the three form a continual learning system.

I think this is closer to the shape of a real product than simply discussing whether "the model should update its parameters". A real personal agent needs both strong general capability and local long-term personalization. It needs both fast memory and slow internalization. It needs flexibility, but also auditability and rollback.

## 8. A Possible Research Question

If I turn the above idea into a research question, I would phrase it like this:

How can we build a fast-slow continual learning system for personal agents, so that it can quickly adapt to users through an external harness, while migrating long-term stable and validated user patterns into an on-device small model at low frequency, and using a cloud large model for memory auditing, data distillation, and complex reasoning?

This question contains several concrete challenges.

First, memory governance. How should the system decide which raw trajectories should be preserved, which can be deleted, and which can be abstracted? How can it avoid the incorrect grouping, overgeneralization, and memory erosion described in the Faulty Memory paper?

Second, abstraction gating. When should experience become a skill? When should original evidence be preserved? When should the system refuse to summarize because the evidence is insufficient or the task structure is unclear?

Third, fast-slow migration. Which content should remain in memory or skills, and which content should enter the on-device model? This boundary matters. Temporary preferences should not enter parameters. Long-term stable patterns are what deserve slow learning.

Fourth, cloud-device collaboration. How can a cloud large model help an on-device small model learn without directly exposing all private data? How can the on-device model filter, compress, and desensitize local experience? How should training data generated by the cloud model be audited?

Fifth, evaluation and rollback. After the on-device model updates, how can we judge whether it truly understands the user better, rather than overfitting to incorrect preferences? If performance becomes worse after the update, how can it be rolled back? How can external evidence be preserved to explain why the model changed?

These questions connect three research lines: agent memory, agentic RL or harness optimization, and on-device personalized models. Each line already has some research on its own. But combining them into a reliable continual learning system for personal agents remains a large open space.

## 9. Conclusion: Continual Learning Is Not "Remembering More", But "Changing Oneself Reliably"

These papers have gradually led me to one judgment: if future agents are to be truly useful, they will need continual learning. But continual learning is not simply remembering more. It is not automatically summarizing one lesson after every interaction. It is certainly not frequently fine-tuning model parameters.

The real question is how an agent can change itself reliably.

XSkill offers an optimistic direction: through an external knowledge base of experience and skills, an agent can benefit from past tasks without updating parameters. Faulty Memory reminds us that external memories may degrade if they are continuously abstracted and rewritten by LLMs, so memory governance is necessary. Although FST mainly discusses LLM post-training, it provides the conceptual framework of fast and slow learning, allowing us to view the agent harness as a fast learning system and the model or on-device personalization layer as a slow learning system.

I now lean toward the view that a long-term personal agent will be neither just a memory system nor just a local small model. It will be a layered learning system. Raw episodes serve as evidence. Abstract memories serve as editable knowledge. The harness serves as the fast adaptation layer. The on-device small model serves as the long-term personalization layer. The cloud large model serves as teacher and auditor. Only such a system may be both flexible and reliable: able to adapt quickly to the user without being misled by faulty memory.

If the old question was "can agents remember?", the next truly important question should be:

How can an agent, over long-term use, gradually become a system that understands the user better, while carrying evidence, audit mechanisms, and rollback paths along the way?
