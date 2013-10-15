---
layout: post
title: "JavaFx: Structuring your Application - View Layer"
date: 2013-10-15 22:32
comments: false
categories: 
---

This post is part of a series of blog posts about structuring your JavaFx application.  
In this post I'll explain the components of the view layer (View and Mediator), how they should interact and some best practices on how to use them.  

<!--more-->  

Here's an overview:

1. Overview
2. View layer (this post)
3. Application logic layer
4. Service - Application State layer

### The View
This class is where you'll use the JavaFx components to build your UI. It's also responsible for displaying notifications, opening new windows or just about anything that it related to the UI (and which should happen on the UI thread).  
I prefer to create my UI with pure Java code instead of using FXML. It's fast, easier to use with Guice and it's giving me more control over my code. Therefore my structure is based on this.  
To keep the Views focused on their tasks, you should follow these rules. 

#### RULE 1: DON'T LET YOUR VIEW EXTEND JAVAFX COMPONENTS
The View should never extend a JavaFx component, it should only use JavaFx components. This rule is also known as "composition over inheritance".   
Our View classes will have a getView() method which will create and return the top-level component/container of that view. That way the actual view is only instantiated when the getView() method is called. This has several benefits:  

1. Remember from the previous post, the View classes are mapped in Guice and injected in the View classes where they will be used. By not extending a JavaFx component, the cost of creation will be lower when Guice creates and injects these Views.  
2. You can extend any other class you want because the design doesn't force you to extend JavaFx components.  
3. The API of your view will not be cluttered with all the methods of the JavaFx component 
 
#### RULE 2: THE VIEW SHOULD NEVER CONTAIN APPLICATION LOGIC
The view can contain view logic (like when to play an animation), but should never contain application logic.   
Application logic can sometimes take up several 100ms. If you do this on the UI thread, then during this time the UI will not be able to update anything, so it will feel unresponsive. Therefore it's very important to run your application logic on a separate thread (which in this structure will be in a Command).  
If some application logic should be executed in response to a button click, it should call the mediator to delegate this to the application logic layer.  

#### RULE 3: PUT YOUR VIEW BEHIND AN INTERFACE
You should create an interface for your View class. This interface should contain the methods that can be called by the Mediator.   
Creating such an interface has several advantages:

1. You create an API for your View which clearly indicates what it can do.
2. The mediator can work against this API, so it isn't tied to a certain View implementation. This is especially handy when writing unit tests for your mediator because you can create a mock for the view and inject that into the mediator.

#### HOW IT LOOKS LIKE IN CODE
Now how does this look like in code when you apply the rules above? I'll show you some code of a view which shows a list of events. The user can select one of the events by clicking on them. 

First we have the View interface:

	public interface IEventListView {
	 
		/**
		 * Asks the view to update the list of events with this new list.
		 */
		void updateEventList(List<EventVO> events);
	}
{:lang="java"}

And then we have the View implementation:

	public class EventListView implements IEventListView {
	 
		@Inject
		public IEventListMediator mediator;
	 
		private ListView listView;
		private ObservableList events;
	 
		public ListView getView() {
			if(listView == null) {
				events = FXCollections.observableArrayList();
	 
				listView =  ListViewBuilder.create()
						.items(events)
						.onMouseClicked(new EventListMouseClickedHandler())
						.build();
				listView.setCellFactory(new EventListItemRenderer());
	 
				//when the view is created, ask the mediator to load the events.
				mediator.loadEvents();
			}
	 
			return listView;
		}
	 
		@Override
		public void updateEventList(List<EventVO> events) {
			this.events.clear();
			this.events.addAll(events);
		}
	 
		private class EventListMouseClickedHandler implements EventHandler<MouseEvent> {
	 
			@Override
			public void handle(MouseEvent mouseEvent) {
				mediator.updateSelectedEvent((EventVO)listView.getSelectionModel().getSelectedItem());
			}
		}
	}
{:lang="java"}

This view can then be used in the overview View like this:

	public class EventOverviewView implements IEventOverviewView {
	 
		...
	 
		@Inject
		public EventListView eventListView;
	 
		public Pane getView() {
			if(view == null) {
				view = PaneBuilder.create()
						.children(
								eventList = eventListView.getView()
						)
						.build();
						  
						...
			}
	 
			return view;
		}
	  
		...
	}
{:lang="java"}

### The Mediator
The Mediator is the postman between the view and the application logic. It is receiving messages (calls) from the View and it passes them on to Commands. It also passes on messages that are coming from the application logic to the view. Again, some rules apply here. 

#### RULE 1: THE MEDIATOR SHOULD NOT CONTAIN ANY APPLICATION LOGIC

The mediator runs on the UI thread, so it should not contain any application logic as this can make your application unresponsive. It should instead create a command (by using the CommandProvider class) and start the command which will then execute the application logic.  
By following this rule we avoid duplicate code and make sure that application logic can always be found in one layer instead of being scattered in multiple layers, which in the end makes your application more maintainable.  

#### RULE 2: THE MEDIATOR SHOULD HAVE AN INTERFACE

The Mediator should have an interface which defines which actions it can perform. These methods can then be called in the view.  

#### RULE 3: DON'T DO COMPLEX MANIPULATIONS ON YOUR MODELS IN YOUR MEDIATOR

In some cases it can be handy to inject a Model into the mediator to directly access the application state without having to write a command for it. But to avoid excessive use of this, you should only listen to changes on the model, get simple data from it or set simple things on it. If the data needs to be manipulated in any way, don't access the model directly but create a command instead which will contain this logic. So be very careful when accessing the Model from your Mediator. 

#### HOW IT LOOKS LIKE IN CODE

When you apply these rules it will look like this. 

First we have the mediator:

	public interface IEventListMediator {
	 
		void loadEvents();
	 
		void updateSelectedEvent(EventVO event);
	}
{:lang="java"}

And then we have the Mediator implementation:

	public class EventListMediator implements IEventListMediator {
	 
		@Inject
		public ICommandProvider commandProvider;
		@Inject
		public IEventSelectionModel eventSelectionModel;
		@Inject
		public IEventListView view;
	 
		public void loadEvents() {
			LoadEventsCommand command = commandProvider.get(LoadEventsCommand.class);
			command.setOnSucceeded(new LoadEventsSucceedHandler());
			command.start();
		}
	 
		@Override
		public void updateSelectedEvent(EventVO event) {
			eventSelectionModel.setSelectedEvent(event);
		}
	 
		private class LoadEventsSucceedHandler implements EventHandler<WorkerStateEvent> {
	 
			@Override
			public void handle(WorkerStateEvent workerStateEvent) {
				view.updateEventList((List<EventVO>)workerStateEvent.getSource().getValue());
			}
		}
	}
{:lang="java"}

### Conclusion
In this post we've covered the View layer and the rules that should be followed. It also showed how the View and the Mediator communicate with each other. In the next post I'll cover the application logic layer were we'll see how heavy application logic will be offloaded to a separate thread in a consistent way.
