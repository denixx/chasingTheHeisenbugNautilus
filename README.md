## Chasing the (heisen)bug

English translation is here [Eng](README.md), [or as a blog post](https://dnxb42.blogspot.com/2026/05/chasing-heisenbug.html)

Українською текст є тут [Укр](Chasing_the__heisen_bug_Ukr.md)

The whole process took a while. Three weeks. Now there is a fix for this issue, and the process of adapting the updated code to Ubuntu 26.04 LTS is in progress. It was released in April 2026. It is also worth noting that a bug was a problem earlier for users of Rolling Release Linuxes, who used GNOME as their desktop. The bug seems to have been in the code for about a year and a half.

It was interesting to complete this investigation and share this experience, first of all, with friends, and, secondly, with colleagues who are "given" these tools, and whom I can help to master AI as a tool faster.

So, if you still have the question, "What part of work can be delegated to AI" if you work in IT?

For May 2026 the answer, without access to "Superior" models like Mythos (does it exist?) or something similar from other providers, and without an unlimited subscription, not much can be delegated. Should AI write code? No. But work is already underway on this, and it seems that for those who can already write/edit, and, most importantly, read code, the main duty of AI for you will be "Rubber Duck on Steroids". Should you give the model work that you cannot verify? Absolutely not. By doing such things, you are effectively investing in a false result. 😊

The point I am trying to make is that you shouldn’t rely on AI as the one that should think for now. Absolutely not.

Why is that? I will tell you what I have learned during this period and, perhaps, tell a little secret behind the message "ChatGPT can be wrong. Check important information" right below the text input field on the relevant site in the context of what this means for you as an IT professional.

## Plot

I was configuring the AI-IDE in Ubuntu 26.04, which was released just a couple of weeks ago. Even before its release, I had invested some time in fixing the Ukrainian translations in the installer of this system. And so, I used the alpha and beta versions for a little longer.

While setting up the AI-IDE in this newly made OS, I noticed that I didn't like my interaction with the Files application. It crashed in response to my actions so often, about 3-4 times during the period of placing old files from the old beta in the appropriate places that I realized that something was wrong.

Well, I think I'll try to figure out what it is and what, most likely, will bother me in the future on my work laptop, when the 26.04.1 release comes out. After all, I am most likely to have the same problem there.

"I already have a personal subscription to the AI-agent-in-IDE, let’s use the communication with it to the benefit of the open source project", I thought.

Sounds quite nice, but only as long as you use it to figure out what's going wrong and to improve your understanding of the problem. And then show or tell the maintainer of the project about those findings.

Why so: [https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy)

I'll try to explain why this has become the norm for the GNOME community and why you probably won't need AI for anything more than a better understanding of which test case leads to an error and whether the same error recurs when repeating the same action.

What you definitely shouldn't leave to an AI agent is to give it the reasoning process. Especially, in those areas where your knowledge is poor, because, as written above, it is up to you to verify what the AI is generating. If you can't distinguish where exactly the AI is telling you nonsense, you are doomed to lose the quality of future actions and thoughts related to the result of this generation. You will need someone who is a carrier of domain knowledge in this area, and, let's be honest, this carrier of knowledge will need to reread the whole text of the entire context window that you fed the AI to make sure that you didn't at least make a mistake in its content. Based on this, in the present conditions, *your* current interaction with AI without access to Superior AI models - is excessive for this domain knowledge carrier. As AI tends to hallucinate where it lacks cheap access to context filling to generate for you an absolutely deterministic (holistic, true and repeatable) answer without errors. By this I mean that having a subscription for $20, you shouldn’t expect AI to work hard and do something worth $2000 for you. There are some response time limits, context window limits, and other limits. AI models have learned from people and they can decide when to be lazy and give you fictional data in an answer. Filling the gap in the context with truthful information, AI would do the amount of work that is irrelevant to the request.

So, a $20 AI is lazy.😊

Yesterday I saw this discussion on Reddit and I liked one of the comments: [https://www.reddit.com/r/webdev/comments/1t6eupy/comment/oki97n4/](https://www.reddit.com/r/webdev/comments/1t6eupy/comment/oki97n4/) . That's a very good idea. Why give you access to the Mythos-level model if you just want to ask how Present Continuous is different from Present Simple? Let's jump straight to the following phrase: "Will you get access to the conditional Mythos for just $20?". Are the developers of such a model ready to provide you with access to it? Are you Lawful Good at Baldur's Gate? If there is a choice between granting access to the conditional Mythos to you or the maintainer of Firefox to clean up vulnerabilities in the code of this browser, who would you choose? I think the maintainer of Firefox is a stronger choice. It is a person who develops a product that has public significance and provides cyber protection to a beneficiary country?

(It might be even Mr. Linus Torvalds himself who is reading this. Mr. Linus, you are the exception. I'm pretty sure you were one of the first to be granted such access.)

So, back to the question, will you be able to get a really smart model with a $20 subscription that will have 100% correct answers and generate flawless code?

No.

Would you like to pay more?

No.

In fact, models have a time limit to give an answer. These models are lazy – do not perform unnecessary actions that will cost too much. They can simply take the content from the "very compressed general knowledge already available in the neuron-model" and provide some hallucinated outputs. You should check it and ask for clarification in the next request if you don't like this part of the answer...

Obviously, we are unlikely to make use of code generation for $20. It’s just a rubber duck on steroids that can guide you. No more.

Do you remember my case of trying to find where the error is in the code with the help of AI? I asked the AI how to fix the app and tried to give the result to a maintainer of the Files application as something that he, a poor guy, will have to deal with... well, that's rude, as they say on Reddit. 😊

Khalid, please accept my apologies.

At the same time, the bug fix was one additional line in the code of the library used by the Files application itself.

This fix was the result of live interaction with a maintainer of this project, brainstorming possible reason for the crash, and cost the great deal of thought. It took two weeks and a half.

In the end, when I was able to figure out how to reproduce the bug quickly on my equipment I invited a maintainer to observe this phenomenon. We tried options when the application crashes. We found out when the app was stable. His mind was stuffed with this for a week.😊

As a result of live interaction, rather than AI’s outputs, this bug has been fixed and will no longer be a problem for me working in a new OS. Also, it will no longer be a problem for anyone else.

When interacting with AI, it’s vital to know the subscription level of the person asking. If it is only $20 or $40, then it will not be possible to get many "insights" from AI. Personally, I will resort to it only when I am stuck with something. Or ask it "here is an application on Spring Framework 3, I want to convert it from war-packaging to executable jar with Spring Boot 3.5. Make me a plan so I can move in this direction."" I’ve already done this twice, and it worked out better than thinking about it all myself.

What am I talking about? An improved rubber duck still won’t be able to write code for you. You can’t tell an agent: "My Files app has disappeared from the screen. Can you fix it?" It is a futile hope of finding a solution. You’d better talk to a real person to find a solution. You can’t come to someone with a copy-paste from an AI conversation. It is disrespectful to the recipient of such a message as they will then need to reread the content of the entire context. I.e. all the questions and answers that you provided to the agent to understand what exactly and how to interpret it.

Yesterday, I reread my chats and downloaded them using the AI chat transcript export function in the IDE. It took me quite a long time to reread them. AI likes to invent words that don’t exist. What kind of code could we be talking about? My favourite one is the phrase "мікро-дриснути" ([search here](ai_chats_exports/ai_nautilus_agent_chat_export_1.md)) in the file ai_nautilus_agent_chat_export_1.md.😊

To sum up, you should:
- Cultivate interaction with people.
- Check the answer.
- Make your own conclusion, even if it can be wrong.
- Tell another person about your conclusion. For example, I know how to crash an application in 12 seconds, but I am not yet sure how exactly to do it.
- Never delegate the process of thinking to AI.
- Never tell other people the AI answer as if it is 100% correct.

If you make a list of what you can:
- Ask some simple questions, such as, "How is Present Continuous different from Present Simple?" - yes!
- "I'm stuck, I'm doing this and that, but I get this error, what could it be?" - yes!
- "Look at this project, build a plan to migrate to Spring Boot 3.5 , make a plan to `upgrade.md` file" - yes! And then doing the work according to this plan is much easier when there are such adored AI lists, like the one you are reading now. This one is made by a human.
- "Look, AI wrote me a piece of code, but it doesn’t work well ..." - a redflag . No! You didn’t check the machine’s output, and no one would like to deal with it, because that would mean that someone would have to figure out what you fed the machine (check the context).
- Run brainstorming sessions like the one you're reading the results of now - yes! AI makes it much easier to tinker with what you're doing for the first time.
- To give AI all your personal accounts and say "do the work for me" - no. Moreover, it is very dangerous from the point of view of maintaining the security of IT processes.
- To give AI some kind of full access to the system and say "do whatever you want, but fix the crash in the Files app" - no.

This list could go on and on, but it is already the fifth sheet of this "report", and I think I should stop.

## Conclusions

The conclusions I made for myself throughout the entire chain of interactions was to stick to human intelligence and not to expect AI to do something big for a small amount of money.

I came to understand that, probably, for the amount of money we are allocated for our corporate subscriptions, we will only be able to do something big once a month.

It would be helpful to give a Gary Bernhardt thread here. [https://x.com/garybernhardt/status/1631866199515738113](https://x.com/garybernhardt/status/1631866199515738113)

Or rather, this humorous response from Ryan Cavanaugh on Gary's thread. [https://x.com/SeaRyanC/status/1631910395790397441](https://x.com/SeaRyanC/status/1631910395790397441)

"I'm excited to stop writing code and start writing machine-readable specifications of precise intended behavior". ©

I've never seen a more apt joke.

Given the recent news that prices for Advanced models have increased several times, for everyday use we are left with only the "auto" mode for selecting models for agents.

Mass code generation is not available yet. Yes, there is some news that gives hope for our magical thinking: [https://www.linkedin.com/posts/ravid-shwartz-ziv-8bb18761_the-research-community-spent-decades-figuring-share-7458345219030319104-Dbpb/](https://www.linkedin.com/posts/ravid-shwartz-ziv-8bb18761_the-research-community-spent-decades-figuring-share-7458345219030319104-Dbpb/)

"some unknown amount of compute thrown at Mythos vs other models" ©

Nobody will give us that. Don't expect it. Write good code yourself and don't take AI answers outside of your chat with an agent.

The fix for the Nautilus heisenbug was not the one suggested by the AI. The maintainer of the code did not need the AI’s texts. The maintainer had to understand what the problem was and try some of the options by rebuilding the application code directly on my computer. I was able to prepare the system so that it did not have my personal data, plus provide remote desktop login. Relying on his "feelings" and "impressions", the maintainer got insights by interacting with me to confirm the exact changes that had an impact on the absence of the bug. Later he tested the app on another system and proved that this bug existed not only on AMD processors. He understood what the problem was and provided a patch in the code of the library his project uses, fixing the root cause in a way that was a fix without creating additional bugs.

It remains to wait for the update to land in Ubuntu...

Links set: [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297), [nautilus#4035](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035), [gtk#7910](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910), [gtk#8173](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173), [gtk#8191](https://gitlab.gnome.org/GNOME/gtk/-/work_items/8191), [gtk#MR9913](https://gitlab.gnome.org/GNOME/gtk/-/merge_requests/9913).
