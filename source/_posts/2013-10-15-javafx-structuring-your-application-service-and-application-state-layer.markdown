---
layout: post
title: "JavaFx: Structuring your Application - Service and Application State layer"
date: 2013-10-15 22:59
comments: false
categories: 
---

This post is part of a series of blog posts about structuring your JavaFx application.  
In this post I'll explain which classes should be used to communicate with external sources and how we store application state. 

<!--more--> 

Here's an overview of all the posts in this series:

1. Overview
2. View layer
3. Application logic layer
4. Service - Application State layer(this post)

### Service
A Service is used to communicate with an external resource. This can be anything, from a webserver to a printer.   
Every Service should focus on a certain resource or a subset of it (like a Service which only performs contact related REST calls).  
Again, it's also important to put your Service behind an interface. By doing this you can mock it when unit testing commands, but it also allows you to create a stub implementation which you can use while the back-end developers are creating the server code and swap it with the real implementation once they have finished the backend code.   

Such a service can look like this:

	public interface IEventService {
	 
		List<EventVO> getEvents();
		EventVO getEventDetails(String id);
		void acceptEvent(EventVO event);
		void declineEvent(EventVO event, String reason);
	}
{:lang="java"}

And the stub implementation:

	public class StubEventService implements IEventService {
	 
		private List<EventVO> events;
	 
		public StubEventService() {
			Calendar calendarDate = GregorianCalendar.getInstance();
			calendarDate.set(2012, 4, 24, 10, 20);
			Date date = calendarDate.getTime();
	 
			 
			events = new ArrayList<EventVO>();
			events.add(createEvent("1", "First Event", "Paris, France", date, date,
					"The event will showcase the first application"));
			events.add(createEvent("2", "Special meeting", "Manchester, UK", date, date,
					"A special meeting for a special application"));
			events.add(createEvent("3", "Technology meetup", "Brussels, Belgium", date, date,
					"Geeking out with the latest technologies"));
			events.add(createEvent("4", "Designer Conference", "Boston, US", date, date,
					"Various talks by the most famous designers in the industry"));
		}
	 
		@Override
		public List<EventVO> getEvents() {
			return events;
		}
	 
		@Override
		public EventVO getEventDetails(String id) {
			for (EventVO event : events) {
				if(event.getId() == id) {
					return event;
				}
			}
	 
			return null;
		}
	 
		@Override
		public void acceptEvent(EventVO event) {
		}
	 
		@Override
		public void declineEvent(EventVO event, String reason) {
		}
	 
		private EventVO createEvent(String id, String name, String location, Date startTime, Date endTime, String description) {
			EventVO event = new EventVO();
			event.setId(id);
			event.setName(name);
			event.setLocation(location);
			event.setStartTime(startTime);
			event.setEndTime(endTime);
			event.setDescription(description);
			return event;
		}
	}
{:lang="java"}

### The Model
A model is used to store and share state across your application. This allows you to react to certain state changes in multiple parts of your application.  
A model is in most cases just a class with some properties that define the state (e.g. the selected contact, the list of contacts that is shown). In some cases it can contain some basic logic, but this should never be complicated (if it is, then move it to a command).   
In most cases it will be a command that updates the model or takes values from it to return it to the view layer.  
As always, you should put the Model behind an interface for unit testing purposes.   

This is how such a model can look like:

	public interface IEventSelectionModel {
		EventVO getSelectedEvent();
	 
		void setSelectedEvent(EventVO event);
	 
		ObjectProperty<EventVO> getSelectedEventProperty();
	}
{:lang="java"}

And the implementation:

	public class EventSelectionModel implements IEventSelectionModel {
	 
		private ObjectProperty<EventVO> selectedEvent = new SimpleObjectProperty<EventVO>(this, "selectedEvent");
	 
		@Override
		public EventVO getSelectedEvent() {
			return selectedEvent.get();
		}
	 
		@Override
		public void setSelectedEvent(EventVO event) {
			selectedEvent.set(event);
		}
	 
		@Override
		public ObjectProperty<EventVO> getSelectedEventProperty() {
			return selectedEvent;
		}
	}
{:lang="java"}

### Conclusion
This was the final article in the series.   
My goal was to share my view on a fat client architecture and hopefully get the conversation started about client architectures with JavaFx. That way we can improve this architecture or even find (better) alternatives.
