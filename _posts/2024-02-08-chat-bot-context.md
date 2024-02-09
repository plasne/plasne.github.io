---
layout: post
title: Conversational Context for Chat Bots
---

I have had a number of conversations lately about what "context" means for a chat bot. In this article, I will document some of my thoughts on improving the conversational experience with better context. By context, I mean the information that is provided to the chat bot to help shape and ground the response.

I refer to a _conversational experience_ because I think this is mostly relevant for conversing with a bot (multiple turns of a discussion that involves two or more parties exchanging opinions and knowledge) instead of simply a transactional experience (asking a question and expecting an answer). Personally, I believe the future of human and AI communication will tend towards more conversational experiences since those are more human-like, more engaging, and more productive.

These are not fully formed thoughts and I have not implemented anything yet, but maybe this will drive more discussion and ideas.

## 1:1 vs group chat

The first topic I want to explore is the difference between a 1:1 chat and a group chat. In a 1:1 chat, the conversation is between the bot and the user. In a group chat, the bot must be able to distinguish between the different users and their messages.

The first implication of this is when the bot should respond. In a 1:1 chat, the bot should generally respond to every message. In a group chat, the bot should respond to messages that are directed at the bot (e.g. @bot mentions). Beyond that, you might give the bot agency to add to the conversation when it feels it has something valuable to say. As a participant in the conversation, the bot would be listening to all messages and deciding when to respond (more on this in the next section).

## Strategic participation

This section will discuss the idea in the prior section where a bot in a group chat is listening to the entire conversation and deciding when to respond.

Let's start with a scenario. Imagine your team at work is having a conversation and a "assistant" bot is also a participant. The conversation starts with people sharing what they plan to do over the weekend. Lets imagine the bot knows where participants live (more on that later) and has access to weather forecasts. It listens passively to the conversation but at some point decides to interject by telling a team member that there is an 80% chance of rain on Saturday and he might consider changing his plans to Sunday. Then the conversation changes to the project the team is working on. Someone asks where the presentation for next week is. The bot knows the answer and interjects again with a link to the document.

This is a very simple example, but it illustrates the point that the bot is listening to the conversation and deciding when to interject. This is a very different experience than a 1:1 chat where the bot is always responding to the user's messages. This type of interaction is also more human-like and reduces the barrier to benefiting from the bot's knowledge.

The chat bot will be listening more than talking and that means this type of interaction could incur a lot of cost and generate a lot of context. An implementation might include a _filter_ that listens to the conversation and decides when to forward messages to the chat bot for possible response and when to simply journal the messages for future context. This filter would be a very inexpensive Small Language Model that knows what skills the chat bot has and can decide if the most recent message or small group of messages is likely addressed to the chat bot or relevant to the chat bot's skills. When the chat bot is invoked, it should have access to all the messages in the conversation (even those the SML did not choose to involve the chat bot in). It can then make an attempt to add value to the conversation. The chat bot may still decide it has nothing meaningful to contribute and not respond.

## Transparency

Unfortunately, some interfaces for group chats (like Teams) do not show bot participation in the same way as human participation. The conversation might look like it is between Amy, Bill, and Jim when in reality there are one or more bots listening to the conversation. Even if the interface is transparent in what bots are part of the conversation, the capabilities of the bots may not be well understood. This creates a few risks:

- Human participants may not be aware that a bot is listening to the conversation and thereby leaking sensitive information.

- Human participants may not know what bots do with that information they collect. The information could be stored in other data stores, be mined for information, be used to train models, be visible to other parties, etc.

- The bot may be capable of mixing context across different conversations. For instance, if the bot is listening to multiple conversations, it might decide to share information from one conversation in another conversation.

- Bots might take actions on the information it learns. For instance, a bot might decide to send a message to another user based on the conversation it is listening to.

- Bots might leak information to the conversation that not all participants are aware of. For instance, a bot might decide to share private information about layoffs with employees that it knows about trying to be helpful.

- Both bots and humans commonly hallucinate information. However, some humans may trust information from a computer more than they should.

My recommendations for transparency are:

- Bots should be 1st party citizens in conversations the same as human participants.
- Everyone (bot or human) listening to a conversation should be visible in the interface.
- The capabilities of the bot should be easily available to participants.

## Scope

A bot that is going to formulate a message will be given some context via prompts. The amount of context it can be given is limited both by capability and cost. Models have limitations on the amount of context they can accept and even when they can accept large volumes of context, they do not treat all input with the same relevancy.

For a 1:1 chat, the context is generally the whole conversation until the user expresses a desire to change the subject. For instance, the user might ask a question, a follow-up question, and then switch to a completely different topic. This may be explicit (the user says something like "I have a different question" or clicks a button to clear the conversation) or implicit (the user simply asks about something completely different).

A few ideas on detecting a change in subject:

- A Small Language Model could be used to determine if the user has (a) said something that should be interpretted as starting a new conversation (explicit), or (b) said something that is on a topic significantly different from the topic they were discussing before (implicit). This would determine what context to send to the Large Language Model.
- One of the outputs of the Large Language Model could be a "change in subject" flag (using the above methods). Upon receiving that flag in the response, we would discard the results and re-run the query with the appropriate context.

The decision between those methods might be driven by how often changes in subject happen. If it is a regular occurance, it might be cheaper to use the SLM approach.

For a group chat, determining the appropriate context is more difficult. There are unlikely to be explicit signals of a change in subject. It is also more common to move between multiple subjects when dealing with multiple participants.

A few ideas on dealing with multiple subjects:

- Messages could be categorized into buckets. Then based on the message the bot intends to respond to, it would pull the relevant context from the appropriate bucket. These buckets might be based on topic, user, etc.
- An embedding model could be used to calculate vectors for each message. Then whenever the bot is invoked, it would calculate the vector for the message it intends to respond to and use that to pull the related context.

This section contains some ideas, but this is a good place for experimentation.

## Audience

A bot may be able to communicate more effectively if it knows who it was talking to.

If the bot has access to sensitive information, it is critical that the bot understand the audience so it doesn't leak information to the wrong person. Consider how you might want to handle this in a group chat:

- If one person asks for information and is authorized to receive it, should the bot respond even if the other's in the chat are not privy to that information?
- Does the bot ask for permission before sharing information with a broader audience?
- Does the bot only respond with information that is safe to share with everyone?

It is possible that the bot could learn information about users by journaling personal information about them to a data store. Then information relevant to the query could be summarized and provided into the prompt as additional context. For instance, if in a previous conversation, a user mentioned they were a vegetarian, the bot might record that information. Later, if the user asks for a restaurant recommendation, the bot might use that information to recommend a vegetarian restaurant.

In a more advanced implementation, the bot might also be able to communicate more effectively if it knows the audience's preferences. For instance, if the bot knows that one person is a visual learner, it might respond with more images. If the bot knows that one person is a slow reader, it might respond with shorter messages.

## Skills

The entity that a person thinks of as a "chat bot" is likely to be a bot that has access to a bunch of other skills. In the simple case, this could be _tools_ and in a more advanced case might be other _skill bots_. In either case, the chat bot gains capabilities by understanding what skills it can draw from to continue a conversation in a productive way.

Some thoughts on skills:

- You might have a skill bot that can read a conversation thread (out of Teams, for instance) to get greater context before responding. This could be the same conversation that the bot is in now so that messages do not have to be journaled to another system (just keep in mind that journaling might still be useful for converting the data into a more useful format, for example, embeddings). This could be a different conversation for historical context.
- You might incorporate some of the ideas in the scope section to filter a broad context into a more targeted context as a skill bot.
- You might have a skill bot that can extract information from a line-of-business system.
- You might have a skill bot that can provide context about a user (e.g. their role, their location, their preferences). You could even imagine something like a digital twin for a person that the bot is communicating with.
