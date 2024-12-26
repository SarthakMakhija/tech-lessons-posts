---
author: "Sarthak Makhija"
title: "Behind the Mic: My journey to delivering a conference talk"
date: 2024-12-25
description: "
I recently gave a talk with Tittu Varghese on 'Questioning database claims: Design patterns of storage engines' at GopherConIndia24 on 2nd December.
In this article, I’ll walk you through my journey - from brainstorming the topic and building the presentation to running mock sessions
and delivering the final talk. Along the way, I’ll share the experiences and emotions that shaped my path to the stage.
"
tags: ["Talk Preparation", "Questioning database claims", "Design patterns of storage engines", "Public Speaking", "Conference Preparation"]
thumbnail: "/talk_title.jpg"
caption: "Photo by Pixabay on Pexels"
---

I recently gave a [talk](https://www.youtube.com/watch?v=_55OM23zhUo) with Tittu Varghese on "Questioning database claims: Design patterns of storage engines" at GopherConIndia24 on 2nd December.
The idea of the talk was to understand various patterns of storage engines (/key-value storage engines) such as:
- **Persistence** (WAL, fsync) 
- **Efficient Retrieval** (B+tree, bloom filters, data layouts)
- **Efficient Ingestion** (Sequential IO, LSM, Wisckey) 

and then question a variety of database claims like **durability**, **read optimization**, **write optimization** and pick the right 
database(s) for our use case. 

In this article, I’ll walk you through my journey - from brainstorming the topic and building the presentation to running mock sessions 
and delivering the final talk. Along the way, I’ll share the experiences and emotions that shaped my path to the stage.

Let's start.
  
### Picking the topic

### Preparing the presentation

After the topic was finalized, I identified four key patterns of key-value storage engines to focus on in the talk:
- Durability
- Read optimization
- Write optimization
- Transactions

Although I wasn’t entirely satisfied with the names of the patterns, I decided to move forward with them. To organize my thoughts, 
I created a <span style="color: #28a745"> simple template to structure the presentation:</span>

**Pattern name** → **Definition** → **Sub-concepts** → **Questions to evaluate database claims**

I had a great brainstorming session with [Unmesh Joshi](https://www.linkedin.com/in/unmesh-joshi-9487635/) around patterns and questions to 
evaluate database claims. I received valuable feedback on the pattern names, suggesting that they should be nouns. 
Based on this input, I refined the names of the patterns as follows:

- Durability → **Persistence**
- Read Optimization → **Efficient Retrieval**
- Write Optimization → **Efficient Ingestion**
- Transactions → **Coordination**

The template provided a clear framework to organize my ideas and the discussion with Unmesh helped in refining it. To give an example, the 
draft structure for the "Persistence" pattern took shape as follows:

<figure>
    <img class="align-center" src="/talk-preparation.png" />
</figure>

I applied the same approach to all the patterns, and soon enough, my presentation was ready. Initially, I drafted a list of 
"side points to cover" as seen in the image above. However, I refined this list by conducting timed practice sessions. This iterative 
process helped me determine which points were essential to include within the allotted timeframe.

I moved on to the next stage, which was getting feedback on the presentation.

### Getting feedback on the presentation

My presentation started with an examination of various database claims. It then delved into key/value storage engine patterns and a set of 
questions for evaluating these claims. However, I felt a sense of incompleteness, as if a crucial element was missing, though I couldn't 
quite articulate what it was.

I had a call with [Gautam Rege](https://www.linkedin.com/in/gautamrege/) and I received some insightful feedback:

1. Revisit those claims at the end of each pattern
2. Conclude the presentation with some use cases, where the audience can select the appropriate database(s) or key/value storage engine(s).

This was a wonderful call because it gave a clear vision for my talk. I could describe the talk to anyone with the following flow:  

**Understand patterns of storage engines → Question database claims → Apply patterns**.

During the call, I also received valuable information about the timing of the talk - it needed to be completed within 22-25 minutes. This 
meant I couldn’t dive too deeply into any one pattern. Additionally, the use cases at the end of the talk needed to be simple enough for the 
audience to quickly grasp and apply the patterns to choose the right database(s) or key/value storage engine(s).

I went back and made the necessary changes. The presentation felt complete now. The feedback resulted in addition of some very important slides, 
the image of one such slide is shown below.

<figure>
    <img class="align-center" src="/write-optimized-claim.png" />
</figure>

### Practice round 1

With my presentation ready, the next step was to practice (in a closed room :)). Since this was the first practice round, I didn’t focus on 
timing just yet. Instead, I aimed to work on the following areas:

- Listening to myself
- Self-evaluating whether I was able to simplify difficult concepts for the audience
- Focusing on my tone, clarity, and ensuring that I didn’t sound nervous or have my breathing interfere with my speech
- Removing redundant sentences while speaking

I didn’t start by practicing the entire talk. Instead, <span style="color: #28a745"> I focused on one pattern at a time, evaluated my mistakes, and once I felt 
confident with a pattern </span>, I moved on to a few full practice sessions of the entire talk.

I practiced 5-7 times and things started to fall in place. However, there were still two key areas that I had to work on:

1. Timing of the talk
2. Audience engagement

Each time I practiced, the talk lasted between 25-28 minutes, with zero audience engagement. By that, I mean I was not interacting with
the audience at all. This made it clear that I had to create enough opportunities in the talk to engage with them - 
whether it was to wake them up, ask questions, or perhaps even incorporate some humor. 

### Travelling to Jaipur

GopherCon India 2024 was held in Jaipur, and my talk was scheduled for December 2nd. I traveled to Jaipur on November 30th, 
giving me one day to focus on improving the two key areas of my talk. On December 1st, I attended a few sessions and took notes on the following:

- Talk time of various speakers
- Attention space of the audience
- Any issues the speakers encountered with the microphone or presentation screen

I did not see any issues with the microphone or the presentation screen. However, I realized that it was crucial to finish my talk under 
25 minutes to allow the audience time for questions. Another important realization was that without engaging the audience, it would be 
difficult to convey the essence of the talk.

I went back to my room to work on "talk timing" and "audience engagement". Before practicing, I thought of showing my slides to [Aman Mangal](https://www.linkedin.com/in/amanmangal/). 
Aman is a good friend and works at Hypermode. He provided valuable feedback, suggesting that I add slides to explain key concepts 
like B+Tree, Bloom filter, WAL, and others. Taking his advice, I integrated slides within each pattern to provide clear explanations 
of these core concepts. An example of one such slide is shown below.

<figure>
    <img class="align-center" src="/talk-sub-concepts.png" />
</figure>

Now was the time to do a few more mock runs.

### Practice round 2

I decided to remove the "Coordination" pattern from the talk. At first, it felt like I was doing the talk an injustice, but the most
important thing was to help the audience grasp patterns like Persistence, Efficient Retrieval and Efficient ingestion along with the
underlying concepts like WAL (write-ahead log), fsync, B+Tree, Bloom filter etc. I wanted them to realize the value of understanding 
these patterns to effectively question database claims and choose the right database(s).

After practicing again, my talk time dropped to 21 minutes.

It was now the time to involve the virtual audience. To do this, I revisited the slides and looked for opportunities to engage them.

<span style="color: #28a745">I was able to create enough engagement points in the slides:</span>

1. One observation I made on December 1st was that while the speakers were greeted by the audience, the applause wasn’t very loud. 
I saw this as an opportunity to inject some humor and create a brief moment of audience engagement.

2. On the slide discussing claims of various databases and key/value storage engines, I decided to ask the audience to identify the icons.

<figure>
    <img class="align-center" src="/database-claims.png" />
</figure>

3. Based on Gautam Rege's feedback, I had already added a slide at the end of each pattern to revisit the database claims. This time
I incorporated animations in these slides (by hiding the table in the image below) and decided to ask the audience whether the 
listed databases (or storage engines) satisfied the given claim.

<figure>
    <img class="align-center" src="/read-optimized-claim.png" />
</figure>

4. Finally, I added animations in "Applying patterns" slides. I decided to ask the audience to pick database(s) or key/value storage engines
for the given use-case.

I practiced again, this time engaging with the virtual audience. Given the audience was virtual, I decided to add 40-50 seconds on each 
engagement point. I felt confident now. Each time I practiced, the talk wrapped up just under 25 minutes, though it was very close to the
limit.

### Giving the talk

Practice is very different from the actual talk. I knew I had practiced enough, but I was well aware of the fact that I might get nervous on 
the stage. My anxiety wasn't about the audience or the stage, but about the pressure to perform well, deliver my message accurately, 
and truly capture the essence of my talk. So, I wanted to feel a little of that pressure beforehand.

This is where [Tittu Varghese](https://www.linkedin.com/in/tittuvarghese/), an architect at the National Payments Corporation of India (NPCI), 
comes in. I shared the stage with Tittu. He skillfully provided the necessary context for my talk, creating a smooth transition for my 
presentation. Tittu spoke for a few minutes, which proved invaluable. This period allowed me to get comfortable with the stage, observe 
the audience, and build my confidence for the rest of the presentation.

I finally gave the talk :). I believe it went well, and I received positive feedback.

### Introspection

Reflecting on the experience, I realize that while I delivered the talk, its success was truly a team effort. I am incredibly grateful 
to Unmesh Joshi, Gautam Rege, Aman Mangal, and Tittu Varghese, whose invaluable input and support were instrumental in shaping this 
presentation.

The mock practice sessions proved invaluable. Timing myself during rehearsals helped me develop a comfortable pace, significantly reducing my anxiety about exceeding the allotted time.

This experience has <span style="color: #28a745">reinforced the importance of meticulous preparation</span> and <span style="color: #28a745">the value of seeking feedback from colleagues</span>. 
I encourage anyone considering submitting a talk proposal to embrace the challenge. The rewards – sharing your knowledge, 
connecting with the community, and growing as a presenter – are truly immense.

I am deeply grateful to Thoughtworks, NPCI, and Caizin for their invaluable support in making this presentation a reality.

The recording of the talk is available [here](https://www.youtube.com/watch?v=_55OM23zhUo) and the slides are available 
[here](https://github.com/SarthakMakhija/questioning-database-claims-gocon24).

*[ChatGPT](https://chatgpt.com/) helped with the article.*


