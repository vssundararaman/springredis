# Spring Redis Integration

In this article, we will review the basics of how to use Redis with Spring Boot through the Spring Data Redis library.

We will build an application that demonstrates how to perform CRUD operations Redis through a web interface. The full source code for this project is available on GitHub.

What Is Redis?
Redis is an open-source, in-memory key-value data store, used as a database, cache, and message broker. In terms of implementation, Key-Value stores represent one of the largest and oldest members in the NoSQL space. Redis supports data structures such as strings, hashes, lists, sets, and sorted sets with range queries.

The Spring Data Redis framework makes it easy to write Spring applications that use the Redis Key-Value store by providing an abstraction to the data store.

Setting Up a Redis Server
The server is available for free here.

If you use a Mac, you can install it with homebrew:

brew install redis
Then start the server:

mikes-MacBook-Air:~ mike$ redis-server
10699:C 23 Nov 08:35:58.306 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
10699:C 23 Nov 08:35:58.307 # Redis version=4.0.2, bits=64, commit=00000000, modified=0, pid=10699, just started
10699:C 23 Nov 08:35:58.307 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
10699:M 23 Nov 08:35:58.309 * Increased maximum number of open files to 10032 (it was originally set to 256).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 4.0.2 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 10699
  `-._    `-._  `-./  _.-'    _.-'                                  
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                      
          `-._        _.-'                                           
              `-.__.-'                                               
10699:M 23 Nov 08:35:58.312 # Server initialized
10699:M 23 Nov 08:35:58.312 * Ready to accept connections
Maven Dependencies
Let’s declare the necessary dependencies in our pom.xml for the example application we are building:

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
Redis Configuration
We need to connect our application with the Redis server. To establish this connection, we are using Jedis, a Redis client implementation.

Config
Let’s start with the configuration bean definitions:

@Bean
JedisConnectionFactory jedisConnectionFactory() {
    return new JedisConnectionFactory();
}
@Bean
public RedisTemplate<String, Object> redisTemplate() {
    final RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
    template.setConnectionFactory(jedisConnectionFactory());
    template.setValueSerializer(new GenericToStringSerializer<Object>(Object.class));
    return template;
}
The JedisConnectionFactory is made into a bean so that we can create a RedisTemplate to query data.

Message Publisher
Following the principles of SOLID, we create a MessagePublisher interface:

public interface MessagePublisher {
    void publish(final String message);
}
We implement the MessagePublisher interface to use the high-level RedisTemplate to publish the message since the RedisTemplate allows arbitrary objects to be passed in as messages:

@Service
public class MessagePublisherImpl implements MessagePublisher {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private ChannelTopic topic;
    public MessagePublisherImpl() {
    }
    public MessagePublisherImpl(final RedisTemplate<String, Object> redisTemplate, final ChannelTopic topic) {
        this.redisTemplate = redisTemplate;
        this.topic = topic;
    }
    public void publish(final String message) {
        redisTemplate.convertAndSend(topic.getTopic(), message);
    }
}
We also define this as a bean in RedisConfig:

@Bean
MessagePublisher redisPublisher() {
    return new MessagePublisherImpl(redisTemplate(), topic());
}
Message Listener
In order to subscribe to messages, we need to implement the MessageListener interface: each time a new message arrives, a callback gets invoked and the user code executed through a method named onMessage. This interface gives access to the message, the channel it has been received through, and any pattern used by the subscription to match the channel.

Thus, we create a service class to implement MessageSubscriber:

@Service
public class MessageSubscriber implements MessageListener {
    public static List<String> messageList = new ArrayList<String>();
    public void onMessage(final Message message, final byte[] pattern) {
        messageList.add(message.toString());
        System.out.println("Message received: " + new String(message.getBody()));
    }
}
We add a bean definition to RedisConfig:

@Bean
MessageListenerAdapter messageListener() {
    return new MessageListenerAdapter(new MessageSubscriber());
}
RedisRepository
Now that we have configured the application to interact with the Redis server, we are going to prepare the application to take example data.

Model
For this example, we are defining a Movie model with two fields:

private String id;
private String name;
//standard getters and setters
Repository interface
Unlike other Spring Data projects, Spring Data Redis does offer any features to build on top of the other Spring Data interfaces. This is odd for us who have experience with the other Spring Data projects.

Often, there is no need to write an implementation of a repository interface with Spring Data projects. We simply just interact with the interface. Spring Data JPA provides numerous repository interfaces that can be extended to get features such as CRUD operations, derived queries, and paging.

So, unfortunately, we need to write our own interface and then define the methods:

public interface RedisRepository {
    Map<Object, Object> findAllMovies();
    void add(Movie movie);
    void delete(String id);
    Movie findMovie(String id);
}
Repository Implementation
Our implementation class uses the redisTemplate defined in our configuration class RedisConfig.

We use the HashOperations template that Spring Data Redis offers:

@Repository
public class RedisRepositoryImpl implements RedisRepository {
    private static final String KEY = "Movie";
    private RedisTemplate<String, Object> redisTemplate;
    private HashOperations hashOperations;
    @Autowired
    public RedisRepositoryImpl(RedisTemplate<String, Object> redisTemplate){
        this.redisTemplate = redisTemplate;
    }
    @PostConstruct
    private void init(){
        hashOperations = redisTemplate.opsForHash();
    }
    public void add(final Movie movie) {
        hashOperations.put(KEY, movie.getId(), movie.getName());
    }
    public void delete(final String id) {
        hashOperations.delete(KEY, id);
    }
    public Movie findMovie(final String id){
        return (Movie) hashOperations.get(KEY, id);
    }
    public Map<Object, Object> findAllMovies(){
        return hashOperations.entries(KEY);
    }
}
Let’s take note of the init() method. In this method, we use a function named opsForHash(), which returns the operations performed on hash values bound to the given key. We then use the hashOps, which was defined in init(), for all of our CRUD operations.

Web Interface
In this section, we will review adding Redis CRUD operations capabilities to a web interface.

Add a Movie
We want to be able to add a Movie to our web page. The Key is the is the Movie id and the Value is the actual object. However, we will later address this, so only the Movie name is shown as the value.

Let’s add a form to an HTML document and assign appropriate names and IDs:

<form id="addForm">
<div class="form-group">
                    <label for="keyInput">Movie ID (key)</label>
                    <input name="keyInput" id="keyInput" class="form-control"/>
                </div>
<div class="form-group">
                    <label for="valueInput">Movie Name (field of Movie object value)</label>
                    <input name="valueInput" id="valueInput" class="form-control"/>
                </div>
                <button class="btn btn-default" id="addButton">Add</button>
 </form>
Now, we use JavaScript to persist the values on form submission:

$(document).ready(function() {
    var keyInput = $('#keyInput'),
        valueInput = $('#valueInput');
    refreshTable();
    $('#addForm').on('submit', function(event) {
        var data = {
            key: keyInput.val(),
            value: valueInput.val()
        };
        $.post('/add', data, function() {
            refreshTable();
            keyInput.val('');
            valueInput.val('');
            keyInput.focus();
        });
        event.preventDefault();
    });
    keyInput.focus();
});
We assign the @RequestMapping value for the POST request, request the Key and Value, create a Movie object, and save it to the repository:

@RequestMapping(value = "/add", method = RequestMethod.POST)
public ResponseEntity<String> add(
    @RequestParam String key,
    @RequestParam String value) {
    Movie movie = new Movie(key, value);
    redisRepository.add(movie);
    return new ResponseEntity<>(HttpStatus.OK);
}
Viewing the Content
Once a Movie object is added, we refresh the table to display an updated table. In our JavaScript code block for section 7.1, we called a JavaScript function called refreshTable(). This function performs a GET request to retrieve the current data in the repository:

function refreshTable() {
    $.get('/values', function(data) {
        var attr,
            mainTable = $('#mainTable tbody');
        mainTable.empty();
        for (attr in data) {
            if (data.hasOwnProperty(attr)) {
                mainTable.append(row(attr, data[attr]));
            }
        }
    });
}
The GET request is processed by a method named findAll() that retrieves all the Movie objects stored in the repository and then converts the datatype from Map<Object, Object> to Map<String, String>:

@RequestMapping("/values")
public @ResponseBody Map<String, String> findAll() {
    Map<Object, Object> aa = redisRepository.findAllMovies();
    Map<String, String> map = new HashMap<String, String>();
    for(Map.Entry<Object, Object> entry : aa.entrySet()){
        String key = (String) entry.getKey();
        map.put(key, aa.get(key).toString());
    }
    return map;
}
Delete a Movie
We write JavaScript to do a POST request to /delete, refresh the table, and set keyboard focus to key input:

function deleteKey(key) {
    $.post('/delete', {key: key}, function() {
        refreshTable();
        $('#keyInput').focus();
    });
}
We request the Key and delete the object in the redisRepository based on this key:

@RequestMapping(value = "/delete", method = RequestMethod.POST)
public ResponseEntity<String> delete(@RequestParam String key) {
    redisRepository.delete(key);
    return new ResponseEntity<>(HttpStatus.OK);
}