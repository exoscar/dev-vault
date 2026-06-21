1. Why not use Spring Events for everything?[[Spring Events]]
   >
   Spring Events are in-process and non-durable.
	If the application crashes after publishing the event,
	the event is lost.
	For cross-service communication or reliable delivery,
	Kafka/RabbitMQ are better choices.
   
   ### Why do many teams avoid `@ManyToMany`?
> @ManyToMany works for simple associations, but real-world relationships often gain additional attributes such as roles, status, timestamps, and audit information. Once the relationship itself contains business data, a dedicated join entity provides better flexibility, maintainability, and domain modeling.