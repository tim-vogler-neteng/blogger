---
title: Thoughts on how to communicate
date: 2026-04-29
draft: true
---
Hear the problem. 

I put that on a sticky note and attached it my monitor. It distills everything below in a single phrase that I can quickly process when the worst of my communication tendencies try to rear their ugly head. I'm not the worst communicator but I think its important to be thoughtful on how you speak with folks especially in the age of remote work and Slack conversations.

Lets back up a little bit, my current employment situation has given me a bit too much time to think about everything. What can I optimize, where am I lacking and need to improve. One of the things that struck me was how I communicate. How do I show up at work and be the great team mate I want to be. I'm a better communicator than I was when I started but there's lots of room to grow. 

What I consistently done well:
1. Don't be an asshole
	Seems pretty simple right? And it is, but it took some time and reflection for me to get there. Back at DigitalOcean we were having an issue with Juniper's support portal. We had two accounts with device support and users spread across the two. It made it difficult to raise tickets and do all the other fun stuff that is required with large infrastructure deployments. It was annoying, and slowed me down a bit, but in hindsight not a big deal. But, I needed to let our account team know how this was unacceptable and that they needed to do better. I was short and demanding on a lengthy email thread. Not long after they took us out to dinner and I was greeted with, "Ah! The guy that's been really hard on us". I was a bit abashed for being called out because I knew the way I acted was immature and maybe even more so because I was the big man while behind the safety of my keyboard. While I learned a lesson then, I didn't learn the right one. The immediate take away was don't be a jerk to folks you might end up face to face with. That's true and decent advice but the real lesson I should of taken from it is by acting adversarial I strained an important relationship that we were dependent on. I was giving them shit about a problem they couldn't fix. I created resentment when I could have built rapport and laid the groundwork for them to actually *want* to help us. 
2. Listen to people, like actually listen.
	Don't try to solve the problem you think they have, help them solve the problem they actually have. And that requires you to hear what they say and 
Where I'm making improvements: 
3. Take ownership of team issues even if it wasn't your fault. 
	 One of my least favorite things in the world is running a post-mortem. Sitting in front of a bunch of peers talking about how *you* made a bad change and *you* made their life harder. But sometimes you've got to step up and take one for the team. An engineer on my team was cleaning up old config and removing IPs from a vpn allow prefix list. The commenting in our template was sloppy and made it look like four IPs were ascociated with a particular vendor when it was actually only the two immidiately below the comment. The change was made and pushed out quickly to all the edge devices across both active dc's. We lost connectivity to that partner and after some needless back and forth they came to us to ask what was going on. Thanks to show system commit it was easy to map the outage to a particular change. Easy fix. I helped fix it and I took the responsibility for running the post-mortem workflow including running the meeting. And while yes its not something I enjoy doing, I did it. When describing the sequence of events it was "an engineer on the team" made a change. And when it came to talking about what went wrong it was "the team needs to do better" at config clarity and the "team needs" to push out changes more deliberitly. I've learned it matters *how* you do it. It's not 'you broke it,' it's 'we need to do better.' We broke it together, we fix it together. Nothing happens in a vaccum. There were mistakes made that lead up to incidents and someone will eventually stub their toe on it. 
	 
	 I hate running postmortems. Sitting in front of peers talking about how someone—in this case one of my engineers—made a bad call that cost us. But I've learned that how you handle that moment matters. It's not about blame. It's about 'we're in this together, here's what we fix.' When you step up like that, it changes the whole dynamic.
	 
	 "I hate running postmortems. But I do them because someone on my team made a mistake and I'm not going to let them twist in the wind for it. We broke it together, we fix it together."

	"I hate running postmortems. Sitting in front of peers talking about how someone—in this case one of my engineers—made a bad call that cost us. But I've learned it matters *how* you do it. It's not 'you broke it,' it's 'we need to do better.' When you frame it that way, your teammate knows you've got their back."
	
1. Focus on presenting solutions not problems. 
	When I joined Fastly, I thought I was pretty great. I had come from a cool startup where I had worked my way up to become an important contributor on the team. I had grown as an engineer by leaps and bounds. And with that new found confidence I went into a large team meeting maybe 2 months or so after joining and made my opinion known about every fault we had. "Why are we doing it that way?" - A gotcha dressed up as a question. It was clear from my tone and the way the question was framed that I didn't think much of what was there and that it should be obvious to everyone there. And yea it was a pain point that needed to be addressed but the way I came at it was adverserial which resulted in a difficult relationship with one my coworkers for my entire tenure at there. Its good to add opinions as healthy discussion can lead to better soltions. But thats predicated on a solid understanding of the problem and most importantly intellectual curiosity about the current solution and what were the constraints on which it was built. 
New Ideas that I want to start implementing: 
2. It's us vs the problem as opposed to you vs me.
3. Ask for help or clarity instead of assuming incompetence.
