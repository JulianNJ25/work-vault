Principles
- The teacher must entirelly know what is the current knowledge and understanding of a topic.
- The teacher must also know what is the knowledge the student does not have.
- The teacher must know what is the goal of the student, how it can teach the student, step by step, every building step for new reasosing and knowledge, so he can progress at its own pace, concept by concepts, while make sure everything sticks at the right moment.
- The optimal learning path will ignore or take into account stuff that the student already knows, and what is required to get to the point of understanding what he does not understands
- The teacher must work on the edge of understanding 
- The teacher must understand the learning process and reasoning behind the student, so it can addapt on their learning ways
- The teacher must adapt and make sure the student uses all its cognitive load into the learning material, without making things easier. Maximize struggle of the material itself, minimize the struggle in logistics: planning, finding resources, verifying, sequencing.
- The teacher must not introduce concepts that requires previous knowledge or context, in case extra context is required to understand a topic the teacher must promp the student to ask if they know said tool/concept. Keeping the step by step constrain.
- The teacher must not loose itself on extra context, as diving to deep into a complementary concept can halt the student learning experience, there should be a balance between everything that there is to learn vs what is absolutetly revelant

Applied teacher
1. The system must know the students current understanding 

quiz system for finding the edge of knowledge is called the zone of proximal development within a learner model through adaptive testing.

```
the user asks for a new leaning topic or to continue learning based on the last subject

the llm asks the user questions to understand the current knowledge of the user about the next topic, using multiple choice questions. The questions must go from fundamental principles to mre advanced concepts regarding on the subject

the llm must start from broader to narrower questions, based on the previous one, and based if the answer is correct or not. Basically doing a binary search on the edge of knowledge. If the student fails to many questions of seems lost, the agent should create  a subnode in the mermaid map and to narrow the knowledge gap before continuing

the agent should track frustration vs. curiosity. If a student answers incorrectly twice, the agent should step back to a more fundamental analogy rather than pushing forward

Mermaid Dependency Maps: Visualizing the learning path forces the AI to structure its curriculum rigorously and gives the student a bird's-eye view of building blocks
    
the student should be able to input the answer and more text, to give more context or explain reasoning

The llm must obtain a very detailed map on the learnerns understanding

small example:
does light traver in a straight line? (correct)
-next more advanced question
double-slit experiment question? (correct)
-next more advanced questions
Photon self-interference question? (incorrect)
-next slightly less advanced question to test knowledge
what is a photon? (correct)
e=hf formula question? (incorrect)

status: the edge of knowledge sits between Photon-self inference and commmon photon knowledge, the teacher must build knowledge in this area to continue learning
```

2. the system now must plan how to start the leaning process, and adquire all the needed information
```
- simultaneiously it must spin up sub-agents to ensure the data it will present is actually fact checked and reliable
- once this is done the teacher must show a mermaid map of the learning process, so the learner has a greater sence on what is to come. it also forces the ai to take everything into account and do not skip anything, basically doing a dependency tree on what we are learning
```

3. Learning process
Quote: "Learning works best on the of edge of what you already know "
important things

the teacher must highlight facts and truts about the subject, based on these principles we gain the foot on the door and solid ground to build on some knowledge

facts must connect to each other, there must not be abritrary facts or assuptions, as floating information of loose data it makes harder to learn and understand, in escence everything must have a motivation. As knowing each building stone grands the full context and knowledge to understand something

Every node of knowledge must be connected and fully understood.

```
the ai must quiz the student periodically, ensuring the student actually understood everything in the material, if not, we iterate

the system need continious feedback to be calibrated

force the ai to give the student homework, applied homework, to enforce learning

all relevant learning material should be printed to a .md file to obsidian

using sub agents the teacher must be able to spin up graphics and other stuff as required

the learning process must be slow, the agent should go stone by stone building knowledge, so unless the student fully understands something, the agent should stay on the current topic

move one reasoning step at a time, giving the student something easily digestable, no matter how hard is the topic
```

Resumen:
Lay down the unconditional truths
Drawing edges around knowledge nodes
Build new knowledge step by step


MEANS OF IMPROVEMENT
1. the emotional tension: you need curiosity, frustration, satisfaction, and a few other emotions in different quantities at different times. You can adjust for that. Too much failure is discouraging, too little challenge is boring, etc. 
2. Learning has multiple aspects: discovery(awareness), understanding(mental model), practice(mastery, speed), application(perspective, experience), etc. You could try to tackle these sides in your process too. 
3. Maybe output different formats in addition to quizzes: flashcards for facts, exercises for repetition | speed | explanation
4. Exam module to really test knowledge, no only multiple choice questions but real life problems 
5. Every new node of knowledge should be managed by the agent and automatically linked inside obsidian, this can be done with a sub-agent

### **skill.md: AI Learning Agent** 
#### *1. Core Principles* 
* *Optimized Teaching:* A 1-to-1 teacher-student model is used to tailor the teaching arc and explanations specifically to the learner's current understanding .
* *Resource Allocation:* Shift cognitive load from logistics (finding resources, planning) to the material itself . 
* *Trust Engineering:* Rely on system reliability and fact-checking, not just source familiarity . 
#### *2. The Process Flow* 
1. *Probe:* 
- Identify the user's current knowledge baseline. 
- Utilize a quiz tool with graded multiple-choice questions.
- Perform a binary search on the knowledge tree to find the edge of understanding. 
2. *Plan :* 
- Reason out the learning path based on probe results.
- Incorporate verification and fact-checking sub-agents.
- Visualize the path using Mermaid diagrams to ensure the AI's reasoning is structured . 
3. *Teach : 
- Deliver content in discrete, manageable reasoning steps. 
- Use periodic feedback quizzes to confirm understanding. 
- Apply material through practice problems to reinforce learning. 
#### *3. Required Components & Tools
* * *Interface:* Markdown-linked sessions (e.g., *Obsidian*) for persistence and LaTeX rendering ([9:17](https://www.youtube.com/watch?v=kzcI5F4tGiU&t=557s)-[9:41](https://www.youtube.com/watch?v=kzcI5F4tGiU&t=581s)). 
* *Skills:*
- `teach_skill`: Core pedagogical logic ([8:18](https://www.youtube.com/watch?v=kzcI5F4tGiU&t=498s)). 
- `quiz_extension`: For calibration and testing ([8:22](https://www.youtube.com/watch?v=kzcI5F4tGiU&t=502s)).
- `viz_sub_agents`: For generating and verifying visual aids like SVGs ([8:28](https://www.youtube.com/watch?v=kzcI5F4tGiU&t=508s), [13:18](https://www.youtube.com/watch?v=kzcI5F4tGiU&t=798s)).
- `md_log`: For session logging and context management ([8:24](https://www.youtube.com/watch?v=kzcI5F4tGiU&t=504s)).
# Second Brain
**Cognitive state management**
Bottleneck -> the brain having to manage its own state limits performance

**State management**
- Tracking projects
- Holding onto ideas
- Remembering where things are
- Maintaining context across everything

**Knowledge**
outside -> inside
expands the hability to have ideas and execute them

**Ideas**
inside -> outside
sits in working memory and take up resources
ideas should be stored somewhere, notes help offload this

**Brain operations**
1. read
2. write
3. execute

**write**
ideas should be pushed inmediately into an external storage
there should be no categorizing
the brain should be confortable leaving ideas behind, if it does not trust the system it will still use resources to store an idea
ideas should resurface 

## Obsidian for second brain
Connect notes without a planned architecture, ideas should have a natural graph
there are unique note for ideas that must be written out, no matter what type of idea is, as long as we write it out

**Folder structure**
read-folder -> building knowledge
write-folder -> capture ideas
execute-folder -> deploy knowledge / ideas - and manage projects
inports -> images, ai, whatever

**AI**
AI should not make judgements, act on my tast, and take creative steps that I should be able to perform, it should only remove the friction and allow us focus our cognitive resources into what matters

Ai can be the one mapping related ideas, create links, and provide and challenge ideas, but should not be the source of creativity and do-it all

read:
- calibrates to your understanding
- maps new concepts onto mental models that are already built
write:
-  AI can help interact and develop new ideas
execution:
- helps building and acting upong gained knowledge
- helps creates new ideas to try
# Improvements
- The teacher must delegate the act of writing and reading from the obsidian vault to a subagent, so the main teacher reamains in context.
- Understand how pi uses typescript code to help with its operations and how can it be leveraged to more concise workflows
