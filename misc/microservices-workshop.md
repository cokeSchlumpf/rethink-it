# What's the buzz about Microservices?

Last week I did a half-day workshop about Microservices. It wasn't an introduction to noobs, but I also wouldn't call it an advanced workshop due to the short time-frame. 4 hours are not a very long time to discover topics about Microservices. But anyhow I got it managed. Following you find my transcript and some comments and notes.

![What's the buzz about Microservices](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/001.png)

---

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/002.png)

The workshop was 4 hours long and had three parts: One short introduction which was kind of inspired by Jonas Bonér's talk [Bla bla Mircorservices bla bla](https://www.oreilly.com/ideas/bla-bla-microservices-bla-bla) followed by two practical sessions: One more focussed on thinking, the other one focussed on hacking and doing with IBM Bluemix.

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/003.png)

As an introduction I asked the audience why there are actually so many people thinking and talking about Microservices. I found these 5 points by [Rajeev Sakhuja](https://www.linkedin.com/pulse/5-reasons-why-microservices-have-become-so-popular-last-sakhuja) really summarize the current situation quite well, three points are the most important from my perspective:

* We've got great technology stacks out there today, whether if it's PaaS like IBM Bluemix or Docker or many many other things. All of those are following simple principles:
  * Run your application anywhere independently from it's actual underlying infrastructure. Locally, on a fat server or in the cloud. Nobody cares.
  * Define your infrastructure requirements with code: Versioned, documented, standardized.

* SOA was, or for some still is, a great thing when thinking about decomposing the enterprise application landscape. But for today's requirements with thousands of devices connected within smart business processes and even way more flexible business process where we start earning money with APIs and data collected by IoT devices... SOA is not flexible enough to adopt the application landscape fast enough and to be in line with the latest technologies you may need to use if you want to be part of those new economies. Thus Microservices are a good way.

* DevOps including at least continuous integration and in best case also continuous delivery is a must-have for every IT environment. And it's also the right starting point to develop, test, deploy and run smaller applications without any or small efforts to bring them up to live.

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/004.png)

The quote from Gardner is kind of provocative, true. Actually I think for the next few years there will be a lot of use cases where Microservices might not be the best solution for every IT project. But if we envision our future world: Every application will be a distributed application, distributed on many systems, devices and things. Traditional, in worst case - for god's sake - stateful applications won't fit into that world. We've to rethink our application architectures to be ready: We have to think distributed and functional, finally think about the clustering and concerns of application components. Maybe we should directly think about [serverless architectures](https://developer.ibm.com/openwhisk/)?

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/005.png)

When we're talking about Microservice architectures we aren't actually speaking about fency technical stuff which is absolutely modern. In fact: We're talking about plain old topics. We really have to rethink our nicely designed object-oriented integrated JEE architectures and split them down to the basics: As said before, splitted, independent stateless services will be the future. But it's not only cutting down our current application landscapes into smaller services and put some "great" HTTP Rest communication between them, it's about designing them in a well mannered fashion to fulfill "one thing" and get them working together easily without that they must know each other. Unix' piping still is a great concept - But how does this concept look like on our cloud platforms and cloud-ready architectures?

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/006.png)

Good starting points when rethinking application architectures for Microservices are [Domain-Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) and [Conway's law](https://www.google.de/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwj_3p-bhvzPAhXIQBQKHbZoBgkQFggjMAI&url=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FConway%27s_law&usg=AFQjCNGBAvAqI4ycf9UhQAI0qLkPxHFI4w&bvm=bv.136811127,d.d24). I think those thoughts are the basics before regarding principles like [12-factor](https://12factor.net/) apps or similar.

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/007.png)

I can't repeat it often enough. Please be thoughtful. Everybody who has seen an introduction on Microservices has seen a picture like the one above: "Monoliths are bad, they are failing anywhere and burn down your whole system. You can't scale them! Proscribe those bastards! Do Microservices!" - Not totally wrong, but... pssst... you want to know something? Designing a really resilient architecture isn't so easy just because you're doing Microservices now... If you are just a little bit foolish you might have a Microservice architecture including a lot of fancy technologies, but you may fail even faster and be much less resilient than before due to constraints you bring into your system.

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/008.png)

... If you don't embrace the challenges and constraints of distributed systems and don't design your application in well-defined domains you definitely will fail.

## Practice: Modelling Microservice architectures

To avoid above mentioned traps the first exercise is not about the fancy technology, it's about thinking about architecture.

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/009.jpeg)

This exercise was done on a white board with some cards - Thanks to my lovely girl-friend who created them for me.

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/naive-solution.JPG)

This is the first "naive" solution a team probably designs the architecture. The cool thing about the cards and the whiteboard is that we were able to easily re-arrange cards and symbols to discuss many topics or solutions quickly. Things like

* resilient communication: which technology? how to reduce it? use caching?
* what happens if communication or a single system fails?
* how to do new releases of single parts of the application?
* how can we test a service or the whole system?
* how to realize logging?
* what can we do if a sub-sequent service is not available?
* what about transaction handling?
* and so on, and so on...

Actually one could write many books about those topics, but if you have just a few minutes to discover the world of Microservices it's a good thing to quickly talk about and visualize patterns, methodologies, problems, etc.

Oh, you're wondering what a better solution might be possible?

Regarding the first, naive, solution: It had many big problems. E.g. if the "Products" Microservice fails your whole Online-Shopping process for your customer will not work, since the WebUI is not able to fetch the product data. Ok, no problem... well: we pull the data into our ordering system. Simply synchronize the database or only some data. Great idea, he?

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/better-solution.JPG)

Hmmmmm.... well... but what happens if the customer service also needs some data from Orders and Products, e.g. for a scoring or sending newsletters? Are you migrating each and every database to every Microservice then? And are you doing it real-time? What about inconsistency? Might also not be the best idea... So, Microservices are not that good you think?

Maybe you should just redo your first step? Don't stick to your naive clustering of services. It's not about customers, products and orders. It's about business processes like "Ordering products" or "Delivery of products into stores"

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/final-solution.JPG)

In that scenario you are using the concepts from Domain-Driven design where you've bounded contexts. A product for example is something totally different for each process. For the ordering-process it might have a price, an availability, expected time of delivery, ... but for your goods-receipt-process it has properties like size, store-location supplier and maybe also availability and price (but for the supply chain guy the price is the purchase price!). So actually you'll have a data-model including some combination of the entities, but always in a different context.

But there are still a lot of remaining questions, e.g. regarding consistency and shared data. Let's take the availability of an product as an example: The ordering process as well as the goods-receipt-process will care about that data and both change it. And this information should be as synchronous as possible. There are again several solutions for that. One would be a kind of event-driven system like shown in the picture above.

And again all that stuff is not new: The final architecture reminds me - not by accident - on actor models. Actor models were already discussed in the 60s and are an applied standard in the really resilient world of telecommunication (and many other places as well!).

To summarize: There are a lot of topics to speak about when it comes to such kind of architectures and it's not always the worst idea to go back in programming history to find well working patterns and solutions.

## Let's develop resilient Microservices on Bluemix

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/010.png)

Basic concepts of Microservices can easily be tested and implemented using totally simple use cases like the above mentioned calculation service which uses an addition- and a subtraction-microservice to calculate complex terms. Based on that example one could play with fail-over scenarios, auto-scaling, DevOps-concepts like zero-downtime deployments, canary releasing, consumer driven test specifications, etc.

On my GitHub account I also provide some sources for the examples:

* Simple starter-kit to quick start: https://github.com/cokeSchlumpf/wlp-jee-starter-kit
* A simple addition service which uses IBM Bluemix Service Registry to register itself: https://github.com/cokeSchlumpf/wellnr-calc-sum-service
* A simple calculation service which uses IBM Bluemix Service Registry to discover the addition service and utilizes Netflix Hystrix for resilient communication: https://github.com/cokeSchlumpf/wellnr-calc-service

![Slide](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-10_Microservices/011.png)
