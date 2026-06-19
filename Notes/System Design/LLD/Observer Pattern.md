# Observer Pattern
The observer pattern is a behavioral design pattern where an object(aka the subject, observable or publisher) maintains a list of dependents(Observers) and automatically notifies them whenever there is a change in its state. 
The pattern allows addition/removal of observers at runtime.

![[Pasted image 20260618222408.png]]


Real-life examples of the Observer pattern include:
- Weather Applications: Where multiple devices receive updates from a weather station.
- Social Media Feeds: When we follow someone on Instagram, Facebook, or Twitter, we become observer of their profile. When they post new content, we are automatically notified.
- Subscription Services: YouTube subscriptions, where viewers are notified of new videos, or Content magazine/newspaper/newsletter subscriptions, where publishers send new issues to subscribers.
- Stock Market Trackers: When the price of a stock (or state) changes, the stock's market system (the observable) sends out notifications to all interested investors (the observers).


Observer Pattern has 2 model of operation
Push model - The observable pushes the data it wants Observer to receive
Pull model: Observer hold Obervable reference object. whenever it got to know something update updated, it pulls the data whatever it needs using Observable object.

### Push Model
![[Pasted image 20260618222809.png]]

WeatherObservable
```
public interface WeatherObservable {  
    public void addObserver(WeatherObserver observer);  
    public void deleteObserver(WeatherObserver observer);  
    public void notifyObservers();  
    public void updateWeather(double humidity,double temperature);  
}
```

WeatherStation
```
  
public class WeatherStation implements WeatherObservable {  
    WeatherData data;  
    List<WeatherObserver> observers;  
  
    WeatherStation() {  
        data = new WeatherData();  
        observers = new ArrayList<>();  
    }  
  
    @Override  
    public void addObserver(WeatherObserver observer) {  
        observers.add(observer);  
    }  
  
    @Override  
    public void deleteObserver(WeatherObserver observer) {  
        observers.remove(observer);  
    }  
  
    @Override  
    public void notifyObservers() {  
        for (WeatherObserver observer : observers) {  
            observer.update(data);  
        }  
    }  
  
    @Override  
    public void updateWeather(double humidity, double temperature) {  
        data.humidity = humidity;  
        data.temp = temperature;  
        notifyObservers();  
    }  
}
```

WeatherObserver
```
public interface WeatherObserver {  
    void update(WeatherData data);  
}
```

WeatherAgent & WeatherApp
```
public class WeatherAgent implements WeatherObserver{  
    @Override  
    public void update(WeatherData data) {  
        System.out.println("Printing from AI Agent");  
        System.out.println(data.temp);  
        System.out.println(data.humidity);  
    }  
}

public class WeatherApp implements WeatherObserver{  
    @Override  
    public void update(WeatherData data) {  
        System.out.println("Printing from WeatherApp");  
        System.out.println(data.humidity);  
        System.out.println(data.temp);  
    }  
}

```

Main
```
package observerpattern.pushmodel;  
  
public class Main {  
    static void main() {  
        WeatherStation station = new WeatherStation();  
        WeatherAgent agent = new WeatherAgent();  
        WeatherApp app = new WeatherApp();  
        station.addObserver(app);  
        station.addObserver(agent);  
        station.updateWeather(50,60);  
        station.updateWeather(100,20);  
        station.deleteObserver(app);  
        station.updateWeather(10000,81.65);  
  
    }  
}
```


In the Push Model, Publisher(Observable) sends all required data inside the event.
 Advantages
- Fewer database queries
- Faster event processing
- Better for asynchronous processing
- Event contains a snapshot of data at the time it occurred

Disadvantages
- Larger event objects
- More memory/network overhead

### Pull Model

Publisher sends minimal information (usually an ID). Listeners fetch additional data themselves.
 Advantages
- Small, lightweight events
- Publisher remains simple
- Flexible for different listener needs
- 
Disadvantages
- Additional database queries
- Multiple listeners may fetch the same data repeatedly
- Can reduce performance

![[Pasted image 20260619064950.png]]

In pull model, the observers has the reference of the observable. observer uses what ever data it wanted.


### Hybrid Model
Hybrid Observer Pattern combines Push and Pull models by sending essential data in the event and allowing observers to retrieve additional data when necessary. It balances performance, flexibility, and event size.







> Why use `IssueCreatedEvent` instead of publishing the `Issue` entity?

> Events represent business actions, while entities represent database state. Publishing a dedicated event reduces coupling, avoids exposing internal entity structure, minimizes payload size, prevents JPA lazy-loading issues, and provides a stable contract for consumers.