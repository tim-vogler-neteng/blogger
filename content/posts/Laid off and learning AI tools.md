
I was laid off from Block at the end of February and the reason provided by Jack was AI efficiencies were reshaping how work is done. Bummer. After sitting for a day or so with the existential dread of an AI dominated future, I decided that instead of sitting still and letting that future catch up to me, I needed to grab the bull by the horns and steer where my career is heading. 

Right after the layoff I made a post to appeal to my network to help land a new job. In it I used my best LinkedIn prose and said that its an exciting time to be in network infrastructure with the capabilities that AI is providing. Even though the post was tailored with a singular objective I believed it then and certainly still do. 

Not to be doom and gloom, but AI is coming and it ***is*** going to reshape how work is done. Jack is not wrong about that. Maybe his timing was off and maybe the way it was approached could have been done with a more deft hand, but the premise is not wrong.

I decided to stop sitting still and build something. If AI is reshaping the work, I want to be in front of the change as opposed to reacting to it. As network engineers without robust software development skills, we need to figure out how to utilize these tools to deliver value. Because if we don't, someone else will.

Tools built over the last two months while juggling other stuff. 
 - [atlas-trace](https://github.com/tim-vogler-neteng/atlas-trace)
	 -  A means of programatically interacting with RIPE Atlas.
	 -  Provide a destination IP with different options and get traces back. 
	 -  Future improvement is to add more intelligence to the results. Find common hops or ASNs that indicate where the potential issue might live. 
 - [maintenance-bot](https://github.com/tim-vogler-neteng/maintenance-bot)
	 - Scrapes a gmail account searching for provider maintenances and adds them to a gcalendar. 
 - https://github.com/tim-vogler-neteng/netaudit
	 - Definitely a work in progress. The idea is provide the tool a yaml file detailing a topology, it spins up the infra in container lab, and then uses a mcp to provide natural language queries about the behavior of the container lab. 
	 - In order to actually make it more useful, need to flesh out getting actual config added to devices, and then importantly, allow for config changes to be pushed so we can test what those changes did. 
 -  [dailyfeed](https://github.com/tim-vogler-neteng/dailyfeed)
	 - Scrapes a series of network focused websites for articles that match particular areas of interest and produces a curated list of interesting articles daily.  
 - [blogger](https://github.com/tim-vogler-neteng/blogger)
	 - The framework and data for this site. 
	 - Workflow is Obsidian as the editor of .md files, commit the files directly to the Git repo, and then using Hugo and a Cloudflare integration, push the updated content to a Cloudflare *page* (Also, Cloudflare is pretty awesome with how easy they make this. But should put in some work to make their UI more intuitive)

Now, theres nothing revolutionary here. These are already solved problems with open source solutions that are much more robust and full featured. But that wasn't the point. The point was seeing if I could do it with an eye toward the future. The issue with open source tools is that they were created to solve someone else's problems. Maybe those problems align closely with mine but they're not going to solve 100% of what I'm looking for. What the above tools represent is the ability to quickly customize a solution to your exact environment and remove the particular pain points that you have with your current tooling. 

The folks that will thrive during the upcoming uncertainty and period of change will be the engineers that have a deep understanding of the technical stack and also lean into AI to build the tools that solve the issues in ***their*** environment.  

And if you can't beat em', join em'. More on that later. 

TTL Expired