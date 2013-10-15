---
layout: post
title: "JavaFx: Structuring your Application - Application Logic Layer"
date: 2013-10-15 22:49
comments: false
categories: 
---
This post is part of a series of blog posts about structuring your JavaFx application.  
Because this architecture is targeting fat clients which have all the business logic on the client side it is important to structure this in an organized and efficient way. In this post I'll explain how you can organize your application logic in a way that it won't block the UI. 

<!--more-->  

Here's an overview of all the posts in this series:

1. Overview
2. View layer
3. Application logic layer(this post)
4. Service - Application State layer

### The Command

A command is a stateless object which is executed once and then disposed of. It has a single responsibility, which clearly defines its purpose.  
Such a command can for example have the responsibility to create a new contact, calculate the total amount of money a client has to pay etc.  
This leads to a higher number of classes, but it has some advantages over a controller:  

* A command will have less dependencies because of its single responsibility. This makes it easier to test.
* When using good class/package name conventions it's easier to find a certain piece of logic.

A command can use Services and Models to do its job, but should never reference the mediator. 

#### RUNNING IT IN THE BACKGROUND
Because business logic should run in a background thread to avoid blocking the UI, I'm using a convention to make every command extend from the JavaFx Service class.   
The Service class is used in JavaFx to run some code in one or more background threads. More info on this class can be found here: [http://docs.oracle.com/javafx/2/api/javafx/concurrent/Service.html](http://docs.oracle.com/javafx/2/api/javafx/concurrent/Service.html)   
This fits perfectly in my design because it makes sure that all business logic is executed in a background thread. Because it's an integral part of our application structure developers won't forget to start a new thread because it will be done transparently when creating a new instance of a command and starting it.   

#### HOW IT LOOKS LIKE IN CODE
This is how it looks like in code. The command has one parameter which should be set before starting the command and it has a dependency to the IEventService. When it's finished it return an EventVO instance with all the details of the event.

	public class LoadEventDetailsCommand extends Service<EventVO> {
	 
		@Inject
		public IEventService eventService;
		public String eventId;
	 
		@Override
		protected Task<EventVO> createTask() {
			return new Task<EventVO>() {
				@Override
				protected EventVO call() throws Exception {
					return eventService.getEventDetails(eventId);
				}
			};
		}
	}
{:lang="java"}

And this is how you can use it in the mediator. The commandProvider is something which I'll discuss in a moment, but it basically creates a new instance of a certain command class. When you call the start() method the command will start its execution in a background thread.

	LoadEventDetailsCommand command = commandProvider.get(LoadEventDetailsCommand.class);
	command.setOnSucceeded(new LoadEventDetailsSucceededHandler());
	command.eventId = eventSelectionModel.getSelectedEvent().getId();
	command.start();
{:lang="java"}

And finally the succeeded handler which is just an inner class in my mediator. It gets the EventVO result from the command and asks the view to show the new details:

	private class LoadEventDetailsSucceededHandler implements EventHandler<WorkerStateEvent> {
		@Override
		public void handle(WorkerStateEvent workerStateEvent) {
			view.updateEventDetails((EventVO) workerStateEvent.getSource().getValue());
		}
	}
{:lang="java"}

### The CommandProvider
The CommandProvider is a class which I've created as sort of a Command factory, which uses a Guice injector to produce the instances. This allows us to inject dependencies in the commands. 

The interface:

	public interface ICommandProvider {
	 
		<T extends Service> T get(Class<T> type);
	}
{:lang="java"}

And the implementation:

	public class CommandProvider implements ICommandProvider {
	 
		@Inject
		public Injector injector;
	 
		public <T extends Service> T get(Class<T> type) {
			return injector.getInstance(type);
		}
	}
{:lang="java"}

### Conclusion
In this article we've seen how you can organize the business logic in a single layer and how you can make sure that it won't block the UI. In my next and final article I'll explore the Service and Model classes which allow us to communicate with external sources and store application state.
