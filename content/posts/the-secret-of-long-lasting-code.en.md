+++ 
date = 2023-02-05T03:07:00+01:00
title = "The secret of long-lasting code"
+++

I cannot talk about every software developer in the world, but I imagine that most of us had to deal with legacy applications at some point in our careers.

But, before diving into it, we should find a definition of legacy software, and [Gartner](https://www.gartner.com/en/information-technology/glossary/legacy-application-or-system) says:

> An information system that may be based on outdated technologies, but is critical to day-to-day operations. Replacing legacy applications and systems with systems based on new and different technologies is one of the information systems (IS) professionals’ most significant challenges. As enterprises upgrade or change their technologies, they must ensure compatibility with old systems and data formats that are still in use.

Now, let us review that definition, to understand the assumptions and reasons for software to be considered legacy.

> Software based on outdated technologies, but critical to day-to-day operations.

For me, this is somewhat of a paradox. The software should not use outdated technologies if it is critical to day-to-day operations. So, either both day-to-day operations and the application are old, so the whole thing is old and must be reviewed, or the software took too much time to be developed, so it does not fulfill the requirements anymore.

In the end, this is about how often you review the software requirements. Keep questioning if the way you work is still correct. Or if the technology is still a good fit for your problem. No one should expect that a particular solution will last forever.

> Replacing legacy applications and systems with systems based on new and different technologies.

In my viewpoint, Replacing a system represents the end of the line. It means that someone made a terrible decision of not investing time and money to keep the process fine and tuned. Or someone made a wrong technical/architectural choice. Or someone poorly analyzed the requirements. Ultimately, it is too late to fix that.

> As enterprises upgrade or change their technologies, they must ensure compatibility with old systems and data formats that are still in use.

Well, that is not surprising and is not a problem for legacy applications only. No matter the size of the application or the technology you use, integration is always a challenge. It is more of a coordination issue and requires regular attention. And it can break at any minute, in unexpected places.

Summing up, **Bad decision-making is solely the root cause of a legacy system**. The software is not legacy. The legacy (or the burden) is the mindset that kept the software as it was until it became a legacy.

## Continuous Refactoring / Upgrading

### From a business perspective

The requirements evolve and require regular ongoing review, often needing refactoring to align the code with these changes. That may add overhead to the business analyst, as the new feature needs to be compatible with existing ones. And refactoring may also increase the risk of introducing defects.

But on the other hand, when we choose a shorter path, to avoid needed enhancements, we may postpone fixes to existing bugs, or increase technical debts, which is also dangerous to the project.

So, to business stakeholders, refactoring is an investment. Well-written stories and well-defined test scenarios will guide the developers in the right direction, and BDD and E2E tests are great tools to detect regressions, so don't be afraid of the new.

### From the development perspective

Writing code is challenging. Enhancing existing code can be double challenging, depending on how far you are from the new requirements. Touching someone's code can be even dangerous, as you may not have that person to guide you. All this will make any developer anxious.

Regular coders will stay in their comfort zone and keep things as they are. Outstanding engineers will take the task but may fail on the first try. However, with enough perseverance, they will excel.

For me, refactoring is about rethinking and learning. It keeps all senior and junior developers up to date about the technology and the business. The more refactoring you do, the simpler and faster you will develop new features.

### Risks of not continually improving your code

Well, your technical debt increases. And technical debt is a concept that refers to the cost of maintaining a software system due to quick and inefficient solutions to problems. It represents the trade-off between short-term development speed and long-term technical excellence. For more details, please follow these links:
- https://martinfowler.com/bliki/TechnicalDebt.html
- https://www.atlassian.com/agile/software-development/technical-debt

### Benefits of doing so

Refactoring/upgrading is not only rewriting code/replacing technology with something else. It is all about stability. Tackling technical debt strengthens maintainability, thread safety, security, scalability, and low footprint. All these are crucial factors for a long-lasting project.

But apart from that, keeping a stable project up and running brings satisfaction, visibility, and recognition. Nothing is more satisfying than a well-done job. And you and your project will become a reference for peers and stakeholders.

### What to do then

Each project is unique, but there are some steps to follow:
- Evaluate your situation and prepare a plan. You want to know where you are and what to do, before starting the journey;
- Break it down into small pieces. It will be easier to get accepted and validated. You can start with single components or fix duplicated code. [Clean Code](https://www.oreilly.com/library/view/clean-code-a/9780136083238/) provides you with many insights;
- Make sure you have good test coverage, it will support you on avoiding regressions. If the test coverage is not enough, [TDD (Test-driven Development)](https://www.agilealliance.org/glossary/tdd) is a good thing to start practicing here;
- Improve your app monitoring and traceability. Not just to prepare for possible new bugs, but also to look at your current situation. For monitoring, I suggest [Prometheus](https://prometheus.io/), For traceability on Spring Boot, [Sleuth](https://spring.io/projects/spring-cloud-sleuth) is the easiest choice. Please, read this [article](https://medium.com/javarevisited/distributed-tracing-in-microservices-spring-boot-125272b58ad8) understand the concept of Distributed Tracing.

Apart from that, there is not much to do than doing your job as it is supposed to be. Excellent coding principles, clean code, meaningful tests, code review, pair programming, Sonar, intelligent IDEs, AI... Use every weapon you may have to ensure the code you write is well written and well covered.

## Conclusion

The golden rule here is: never to stop trying new things, or you are stuck in the past. And most probably, everyone else will waste a lot of time and money doing the same thing you did over and over. At some point, your software will become a legacy system, and everyone will complain. Eventually, it will get replaced.

So, please do not be afraid of changing the code. No piece of code is generic enough to cover all possibilities, nor so well written that is the best solution for the requirements.

**See you | Bis geschwënn | Até mais | À bientôt**
