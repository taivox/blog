The expectations for the speed of software development have risen considerably since the 1980s. Our audience no longer accepts a year of waiting for the next software version, and as developers we need to keep up with this pace. The audience is expecting us to deliver fast and often. To meet these expectations, we need to remove waste and speed up the development process. And the DevOps culture is our best friend for figuring out how to reduce lead time.

For most teams it is obvious that they should optimise and automate processes to reduce development time.  But very often teams focus their DevOps efforts on automation and local optimisation only. While this can put out considerable fires for the time being, it can distract us from our main goal that actually has an impact - reducing lead time.

In order to successfully reduce lead time in software development we need to thoroughly understand how and why we are changing the ways we work. To figure out the impact of our new management practices, upgrades and automation, we need to implement and track DevOps metrics that cover the whole value stream.

Before anything else, we need to thoroughly understand our main goal - quickly delivering value to our clients.

The less time we spend on each delivery, the more value we are able to deliver.

Once the whole team is on board with this concept, you can start taking actions towards actually reducing lead time.

There are three great tools that really improve your ability to reduce production lead time and improve value delivery:

1. Value stream mapping
2. DevOps Performance review
3. Value stream management

The definition of value stream in DevOps is “a set of actions that add value to a customer - from their initial request to final delivery and support”.

#### Value Stream Mapping for Software Development

Mapping the value stream is not about creating documentation for the coach or consultant. DevOps value stream mapping is about taking the time to actually understand what’s going on in your organisation, and it is best done as a workshop with lots of discussion and observations with all relevant team members.

In reality, work practices are rarely ever exactly documented. Things on paper and things in practice are usually different (and both suboptimal). In addition, how people think they work is very different from how they actually work.

The mapping process gets everyone on the same page about what is actually happening, and what can and should be improved. It will discover things that work well and do not need changing, but also items that do not add value. If all key stakeholders are involved in the value stream mapping, everyone will be able to understand and agree on the future priorities.

While there is no single right way for every organisation to workshop through the value stream, there are a few key success factors that definitely help:

1. Trust a strong and skilled facilitator to lead the process
   The workshop leader should be neutral and competent. If there is nobody skilled in this type of workshops within your team, do not hesitate to get external help from a consulting agency. It is even statistically proven that better results are achieved with an external expert. In-house project leads might not be neutral enough, and may be perceived to be pushing their personal agenda.

2. Make sure the leadership takes initiative
   Only the core stakeholders can initiate and actually see through the process of changing the way your organisation works.

3. Large strategy, narrow scope
   Do not waste time with details and processes. Set a clear focus and position your initiative to support strategic goals.

4. Relevant people only
   Choose your mapping team wisely. Make sure all relevant people are present, but if you need to include more than ten people, you’ve probably set the scope too wide.

5. Single timeslot
   Do not multitask over multiple weeks with mapping. Actually take three full consecutive days for the whole team and focus.

6. Brief all stakeholders daily
   Make sure the full leadership gets daily briefings instead of a single final report. While not actively involved, a results presentation is not enough. They also need to be informed of the process.

7. Design for the future
   Value stream mapping is mostly about doubting and giving up your current practices. Do not get stuck in existing things, but rather think about the state where you’re at tomorrow and what are the actually important ways for achieving that.

8. Make an actual plan
   Once you have a description of where you want to go, make sure you have an actionable roadmap of how to get there.

#### DevOps Performance Review

When reviewing your performance, the results from the value stream mapping serve as context. Your next steps could be as follows:

- Go through a DevOps performance review to know where you stand
- Make improvements
- Set up DevOps performance metrics to measure the impact

A great input for a devops performance review is to start tracking the four performance metrics of [DORA](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance) (DevOps Research and Assessment).

1. Deployment Frequency
   High deployment frequency is always better than low. Releasing to production often and in small batches is less risky - it will be much faster to troubleshoot errors after a small change than after a large batch of major changes. High deployment frequency also lowers the amount of partially done work at any given time. The ability to move things forward quickly makes sure that the currently relevant business needs of the organisation are supported at any given time.

2. Lead time for changes
   This is the time it takes a GIT commit to get into production. Measuring this will reveal how fast your team and clients are actually able to get support from technology - and it’s always better for business to get the new features and upgrades rather earlier than later. Therefore it should be your critical priority to reduce production lead time.

3. Time to restore service
   While the lead time for changes usually comes with most pressure from the business side, don’t forget to also measure the time it takes to recover from a failure in production, and the lead time for fixes. The ability to quickly restore services makes your mistakes less impactful, and lets you make changes confidently without the stress and fear of possibly breaking things.

4. Change failure rate
   We should not lose in quality when gaining speed. Measuring the percentage of deployments that cause a failure in production will let us know how much we waste time on reworking things. Fast and excellent is better than fast with failures - and knowing our change failure rate will let us know which of these modes  we’re currently operating.

#### What Causes Long Lead Time

Remember why we’re doing all of this in the first place? Our main goal here is to reduce lead time. Going through the value stream mapping process and doing regular DevOps performance reviews should uncover the causes of long lead time and give you direct insights on what to fix.

Organisations who skip directly to automating processes can create even longer lead times, as very often the actual causes of long lead time are hidden in general team management issues or even misaligned company-wide strategies.

##### Strategic Causes for Long Lead Time

These issues can only be fixed with changing the core ways how your organisation works.

1. Producing extra or unnecessary features
   Overcomplicated solutions and abandoned projects create a lot of code that eventually gets shelved or never used.
   Often this can be caused by a strategic goal of keeping everyone busy instead of productive, or simply by producing features that are not ready to be implemented yet (and will lose importance after actual development needs are discovered). The longer it takes to develop a feature from idea to finish, the less likely it will be relevant by the time it is available.
   As no amount of automation can make sure that only necessary features make it to production, it’s important to invest in ways to weed out initiatives that work on irrelevant things and therefore lengthen lead time.

2. Handoffs
   Having to involve other people and teams when trying to achieve a single goal. Up to 50% of information and knowledge get lost in every handoff, causing additional time and resources spent. To manage work in a more effective way, one should minimise handoffs. The more a single team or developer is able to achieve without getting others involved, the shorter the lead time.

Delays
A major source of time-waste for developers is the need to wait for other people to respond and react before being able to move forward. Removing the need for co-worker contribution and constant approval and verification processes can critically shorten  lead time.

##### Tactical Causes for Long Lead Time

These issues can be addressed locally within the team, without getting the whole organisation involved.

1. Re-work and re-learning
   The amount of time developers spend on re-discovering old code and re-solving old problems can have a huge impact on lead time. This can be avoided by making sure the documentation is up to date, and the previously discovered know-how is easily accessible for all team members.

2. Motion and multitasking
   Having to switch between different projects not only kills the productivity of a single developer, but also adds to the lead times of every single project they are working on. The time spent on re-focusing and re-learning can come from either handoffs and delays, but also from general team management issues where disrupting focused work is allowed.

3. Defects
   There can be many reasons why work needs to be re-done, but it’s clear that tackling challenges twice as were not executed up to standard at the first time will pile up additional lead time. Figuring out ways to lower failure rates should be a critical priority when shortening lead time.

#### DevOps Value Stream Management

We’ve by now learned that value stream management is not just an issue of optimising technical processes or upgrading the management of technical experts. It is only impactful when the initial lead and commitment come from the leadership and C-level management. Getting started with the new mindset organisation-wide is a strong push that needs full support from the leadership, as only they are able to see through actual changes in work arrangements.

Once trust with leadership is established, the ongoing value stream management can certainly become more decentralised and decision-making delegated to the experts and teams. When the leadership has accepted that impactful changes are allowed to be made at the local level, a regular DevOps performance review is a good way to keep them assured that the metrics are on track.

Do not expect value stream management to be a set-and-forget job. You should approach this as a regular routine of continuous improvement - when one shortcoming gets fixed, another one gets prioritised.

There are many tools and value stream management platforms that make it easier to implement the new mindset and processes. They can cover parts of your global value stream and help with metric tracking, but they do not replace the strategic work. You will still need to do the actual work of value stream mapping and implementing your DevOps performance metrics.

Luckily you can always turn to the experts - Entigo is more than happy to help you choose the best tools for your stack. Our experience of conducting value stream mapping and review workshops gives us expertise to support these processes at organisations of many different sizes and growth stages.

Primary goal of DevOps is to improve the product development performance. So in order to evaluate how your organisatsion is doing with DevOps, you should demonstrate how much it helps to improve your overall value stream performance. For DevOps performance review there are good scientifically proven practices in place on what to measure in order to gain a good insight into your DevOps practicies as described in further detail below. Still it is worth noting that before starting finetuning our metrics for DevOps initiatives it is worth taking some time to map out our overall value stream and determine if improvements in DevOps enable us to significantly enhance our performance or the bottleneck is elsewhere in the process.
