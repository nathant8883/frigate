# Real examples

Nathan's own messages. Spelling is corrected, because typos aren't part of the voice. Everything else
is as he typed it: case, missing or kept apostrophes, missing periods, dashes, `...`, haha. The
**Slack** sections are the primary reference. The **To agents** sections show how he states decisions
and asks for things, but they run terser and bossier than he is with people.

## Slack: quick replies (DMs, threads)

Bursts, sent as separate messages:

```
why that frequent?
i think generally we'd prefer 15 minutes over 30 mins
just more real time
```

```
Is gulf a carrier?
or are these their stores
so they are a pretty big client
it would be cheaper on us
```

- we could probably handle the 5 mins, we would just be scaling them pretty high....
- i think 15 would be a nice middle ground
- be sure to let christine know when the rebase is done
- polling the well and the board are in the RC version so tatum can just update her branch and itll have it
- im pretty sure stacking PRs doesnt auto rebase them and stuff it just kinda couples everything
- that google button at the top isnt a part of their single sign on
- Gimmie a few, yeah saw those alerts but they stopped firing, thought it was ok  /  Tank wagons?
- KB-48942: In-Cab Tool Kit  /  fyi looks like some merge conflicts on this but then we can merge it
- tbh thats why i came back to check
- only if i have all my ideas at once :stuck_out_tongue:
- but noted haha
- Ooof sorry bro that's the worst :melting_face:
- Feel better!
- alright now just pray the madness is over haha
- I forgot about recommending a bug to be closed lol
- cool im still making a few tweaks, but im going to do a writeup and yeah I think you trying it out is just as valuable as getting one or two of the juniors involved
- For optimizing turns i think im going to turn that off once the driver starts on the turn, just to avoid any noise on that right now  /  That sound ok?
- whats the reason for this work? <PR link>

## Slack: answering with a decision

- Good eyes. Yes it is a real case, however we are going to make them manage that explicitly in this work <KB-52130 link>  /  Basically the dispatcher needs to explicitly declare there will be no load for this turn for it to be skipped. Otherwise you cant tell whats just a missing decision vs an explicit decision. So yeah that being a blocking error was intentional
- subtask bugs shouldnt have EPs on them. Only the initial work. Bugs are part of that effort estimation so shouldn't be double counted
- id probably rather make some progress towards the long term goal rather than try to do some duct taping - we have plenty of ram to scale this up and keep it stable so we are not really on fire
- All the requests still need to go through the product team - we dont want stuff going directly to tech team. Wesley is a fine point of contact, its also fine to ask for some help in ram rod or something, people are generally pretty willing to hop on and talk out something
- We're just trying to limit distractions on the team - we've always got an engineer working the T3 rotation. It would be helpful to us to keep the distractions focused on them

A numbered answer in a group DM:

```
These are mostly managed by the tech team so they probably just made a mistake.

1. if its in use, just move it into the kustomize.
2. if its not deployed rn just get rid of it
3. Their SFTP drive integration will go down - its either tank readings or their backoffice that run through that file system. This is production though
```

## Slack: pushing back on a design

```
Im not a fan of this strategy for this use case. The problem is those documents are transactional so there are tons of updates that will be continuously happening that the in cab doesnt need to know about. You wont be able to filter those on the server successfully without using a pointer field like version - which means youd have to build a bunch of app code to determine if the change was relevant to the in cab or do a ton of work you dont need to
```

- It's a bit too much like a signals architecture which is kinda an anti pattern for most systems
- you cant use version number incrementing... its circular. All youve done is put the critical write path onto updating version numbers instead of emitting an event
- Let me flip it - without considering the implementation problems, generally this is why this architecture is frowned upon  /  When you do this kind of architecture (signals, db hooks etc). It sounds great because you think "its easy i never have to worry about missing publishing a critical event". However what youre actually doing is creating a hidden control flow..
- Its a very similar reason to why you dont do cross service database linking. You want to keep consumers running off an established API so you can make guarantees about it. When you let stuff just deep link into the database object the consumer becomes coupled to the data model
- (also similar argument id make... you gathered up all the important functions in a few mins, its not hard. There are very few)

## Slack: posts and escalations

```
Hey guys notes from the AI pod today.

Both teams are moving to launching alpha soon.

We'd like to start thinking about the roadmap for Scout. Attached is the short list of ideas that we think would provide value.

Not to add more stuff to yalls plate but Im really looking to see who in the product org wants to be the champion of this and provide some direction for where we go next. Let me know your thoughts
```

```
FYI I finished chatting with Steve and Sandrine about the tweaks to the on call process during business hours.

1. We're going to try to make sure we have a channel that people can be in to have high level visibility of the issues as they get escalated so its less hidden
2. Going to look at options we have in the app for during business hours escalations. Either going to put the T3 person on as the primary or put everyone on so it round robins. Expectation is still relay the issue to whoever is best suited to tackle it during business hours - Support team just needs to make sure there is someone accountable for the issue.
```

```
Hey I heard through the grapevine that the support team is raising critical to the on call person during business hours - My expectation was that the incident IO should only be for outside of business hours and during business hours tier 3 should be handling those items and communicating as needed to pull in extra support from the team?

Only bringing this up because it sounds like its causing a bit of a mess on our sprint and the team doesnt get any visibility
```

```
Just sent everyone here an invite for a TW meeting this afternoon. We want to get started on the in-cab next sprint

Goals:

• Preliminary architecture
• Near term tech plan (stories)
• Changes we want to make to the core architecture of the app as a whole

I want to get a few stories down and earmarked for people to start working. On top of that the sooner we can get the TW work to be able to run in parallel the more people we can put on it
```

- Hey guys - working to get Steves on call plans into effect.  /  The goal for this is to establish what happens when a support person needs to escalate issues after hours and make sure we have multiple people participating in after hours support.
- Hey guys yesterday i merged in some new skills for doing development work. My goal is to outline our entire SDLC and then give us a pipeline we can keep refining. The design of this is to allow you to use them a la carte or you can experiment with the entire workflow.
- hey random - working on the v2 version of the ai chat in S&D. Im supposed to give a demo of it at the retreat. Before i start combing slack or making stuff up any use cases yall would really want to see from it? or know anyone on the team who would have some stuff
- We have a new on call provider we are going to be testing out, im going to send out info today so people can get the app downloaded

## To agents: asks / requests


- can you confirm that if a user puts the pr into changes requested that resets the SLA / it falls off?
- change this to a query so its not pulling all those schedules into memory to check if they have orders
- let me know when its fixed and ill let our consultants know they can resume working
- can you hit the movements api for order 1007 and see if the manual accessorial i added flows through?
- We should wire up the recalc button per Wesleys PR review, generally its a no op but we dont want it to throw an error like it currently does
- have alpha make an artifact of the plan so i can review it easier
- sentry is called mom-sentry-dev-token, mom-sentry-prod-token. both in the gcp dev op vault
- get loves_test pointed at RC so we can see scout fully working from the prod build
- stop trying to open a browser just lmk when the burner is finished and ill vet it

## To agents: decisions / rulings

- table that for now - im happy with the tool, lets roll it out to a few more people. Ben, Jack, Jim, Luke
- november looks good to me - lets put into ready for testing and assign Ben Keener the validate
- what you built works, just change the error to default to 30. that way anything outside is a warning, above 30 mins is an error. works perfectly
- i think unit coverage will be fine for alphas work. and thats wrong you can turn on tolls in the cost books - they will use the rates straight off the route
- we are keeping freight. just need that recalc button thing fixed, once thats done re-request review on Wesley
- dont care about code changing after approval or automation prs, we are already looking in the confines of our sprint work anyway. Also verbiage lets call this Time to merge or something, I dont want QA team to think these metrics are throwing them under the bus
- Id probably say Needs Guidance would be the better verbiage then?
- i dont want to fix the transitions thats coming in different work, but i would like to work on the UI facing elements

## To agents: feedback on work

- comparments changed audit dont show how the products were changed, useless audit
- we need a flag that distinguishes before window vs after window, they are not the same
- the count should be the number of flags in that category so having no count for supply not set doesnt make sense
- ok i see it now, these should have the low designation though, warnings are separate
- its also still clustering too early, i should be able to see all of DFW without clustering
- we dont need these bol ones, they arent about turn state, these are movement audits which go in a different piece of work
- for the last error message id say product cannot be loaded into compartment that last carried {previous} your wording is too casual

## To agents: questions

- is the silence bell at the directive-carrier level or just the directive level?
- can i turn that on for everyone at the app level or is this a per person thing?
- so help me understand the design here.... hes made the itinerary stateful? So a driver is relying on getting these pushes to be up to date, there is no fallback poll or anything?
- whats your concern level if i told you about 1000 drivers would be opening websockets via this method?
- did our solution take into account the must go should go could go priority of the orders?
- ok well 8.50.0 was rolled out to about 6 beta clients and the release panel doesnt show any of them as released
- other people are seeing it but not me.... any idea if i need to do something?
- do we currently get from our dsus from the in cab the arrival lat/lon of the driver? We'd like to get that added to the delivery ticket

## Multi-point messages

He numbers them and keeps each point to one line:

```
1. the pulsing on the map is white so its very hard to see
2. in the order details when clicking the column i dont see the link to order movements
3. the store/terminal markers seem to be rendering behind the route lines
4. the assign all should just say "Assign All" instead of "Assign all N"
```

## Write-up openers

- Here is the goal - we did a prototype of an ai agent called gravichat on the gravitutor demo branch. We spent a lot of time rewriting the core of the agent and have built a new backbone of the agent in this repo.
- We will be experimenting with an auto bug fixing flow. I think generally we need to take a tagging pass to figure out how concrete / complex the issue is. Then we take a pass to do an investigation/triage. Once we have a proposed solution then we can queue up agents to fix issues.
- The goal of the triage template is to give a super concise here is what the customer is experiencing. What the evidence for that is (active issue, sentry, reproduced). And then a tldr high level summary of the fix. In a separate section the actual implementation plan can be spelled out.

His own template shape for a write-up, verbatim structure:

```
1. Customer wants trailing spaces to be removed when inserting markets.
2. Code review verifies this functionality doesnt exist.
3. Proposed fix: add stripping functionality to market update endpoints and v1 apis.

Business logic change
1. Markets will no longer be able to have trailing spaces.

Confidence
High

Technical implementation
...
```

## Before / after

**Agent draft:**
> Hey team! 👋 Just wanted to give a quick update — the PR reminder tool is back up and running after the MOM database wipe. Let me know if you run into any issues!

**In his voice:**
> Hey guys - pr reminder tool is back up after the mom db wipe. If you stopped getting pings lmk and ill get you set back up

---

**Agent draft:**
> Great question! I've looked into it, and it appears the tolls aren't showing in supply options because the cost books may not have tolls enabled. It might be worth checking that setting.

**In his voice:**
> tolls come straight off the route if you turn them on in the cost books. supply options is COST only so thats why they arent showing up - can you check that setting?

---

**Agent draft (write-up):**
> **Executive Summary:** This document outlines our exciting new approach to automated bug fixing, leveraging AI agents to streamline the process!

**In his voice:**
> TLDR - we're piloting an agent flow for prod bugs. Each bug moves through gates (estimate, triage, build) and each gate produces an artifact a person approves before the next one starts. The goal is that the no-brainer fixes get done by agents and the harder ones get deeper human review.
