![LwM2M](http://www.openmobilealliance.org/wp/Images/OMA-129%20Lightweight%20M2M%20Logo_RGB_full.jpg)

# LwM2M Overview

OMA Lightweight M2M is a protocol from the Open Mobile Alliance for M2M or IoT device management. Lightweight M2M enabler defines the application layer communication protocol between a LWM2M Server and a LWM2M Client, which is located in a LWM2M Device. The OMA Lightweight M2M enabler includes device management and service enablement for LWM2M Devices. The target LWM2M Devices for this enabler are mainly resource constrained devices. Therefore, this enabler makes use of a light and compact protocol as well as an efficient resource data model. It provides a choice for the M2M Service Provider to deploy a M2M system to provide service to the M2M User. It is frequently used with CoAP.


# Eclipse Leshan
[Eclipse Leshan](https://eclipse.org/leshan) is an OMA Lightweight M2M server and client Java implementation.

[What is OMA LWM2M ?](http://technical.openmobilealliance.org/Technical/release_program/lightweightM2M_v1_0.aspx)  
[The specification](http://openmobilealliance.org/release/LightweightM2M/V1_0-20170208-A/OMA-TS-LightweightM2M-V1_0-20170208-A.pdf).  
[Object and Resource Registry](http://www.openmobilealliance.org/wp/OMNA/LwM2M/LwM2MRegistry.html).  

Leshan provides libraries which help people to develop their own Lightweight M2M server and client.  
The project also provides a client, a server and a bootstrap server demonstration as an example of the Leshan API and for testing purpose.

## Modules
The eclipse leshan project contain the followind modules

`leshan-core` : commons elements.  
`leshan-core-cf` : commons elements which depend on [californium](https://github.com/eclipse/californium).  
`leshan-server-core` : server lwm2m logic.  
`leshan-server-cf` : server implementation based on [californium](https://github.com/eclipse/californium).  
`leshan-client-core` : client lwm2m logic.  
`leshan-client-cf` : client implementation based on [californium](https://github.com/eclipse/californium).  
`leshan-all` : every previous modules in 1 jar.  
`leshan-client-demo` : a simple demo client.  
`leshan-server-demo` : a lwm2m demo server with a web UI.  
`leshan-bsserver-demo` : a bootstarp demo server with a web UI.  
`leshan-integration-tests` : integration automatic tests.  

# Tutorial
This tutorial presents a simple scenario where a raspberry pi will act as a lwm2m client publishing the temperature and humidity from the DHT11 sensor. The lwm2m server will be also running on the raspberry pi.

The lwm2m client uses the 3303 and the 3304 objects represent the temperature and the humidity sensor.

![LwM2M](http://10.3.2.91/Socialite/lwm2m-tutorial/uploads/d3123a956fb053eb492caa97a5941716/Capturar.JPG)

## Preparing the development environment

1- To start we need to install the [Eclipse EE](http://www.eclipse.org/downloads/packages/eclipse-ide-java-ee-developers/keplersr2) and the [Apache Maven](https://maven.apache.org/install.html).

2- After then, we need to clone the [Leshan](https://github.com/eclipse/leshan) project to the eclipse workspace and run the following maven commands to prepare the eclipse workspace:

```shell
mvn -e -Declipse.workspace=<workspace folder> eclipse:configure-workspace
```
```shell
mvn eclipse:eclipse
```

3- Opening the eclipse we should be able to run the leshan-client-demo project and the leshan-server-demo project.

4- Accessing to [localhost:8080](localhost:8080), a device will appear with a virtualized temperature sensor.
    
## Leshan client code
The leshan client demo available in the leshan repository simulate, by generating random numbers, a temperature sensor and implements all the required fields of the 3303.xml model.
The code is structured in four classes: the LeshanClientDemo class, the MyLocation Class, the MyDevice class, and the Temperature Class.
To implement this tutorial we need to create:
 - A class to collect the humdity and temperature values from the DHT11 sensor;
 - A new class to represent the humidity sensor;
 - Modify the existing temperature class, to get the DHT11 temperature sensor and not random values;
 - Add to the LeshanClientDemo class the new sensor model (3304-humidity sensor);

#### 1- Collect the DHT11 data

Create a class that acquire the DHT11 sensor values by using a [python script0](https://github.com/adafruit/Adafruit_Python_DHT/blob/master/examples/AdafruitDHT.py). 

Obs: There are sone timming problems when using native java libraries to acquire the DHT11 sensor data in raspberrypi. Thus the DHT11 data is acquired using this approach. 

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.Observable;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import org.eclipse.leshan.util.NamedThreadFactory;

public class DHT11Class extends Observable {

    private static String line;
    private static String[] data;
    public static final double minTempValue = 0;
    public static final double maxTempValue = 50;
    public static final double minHumiValue = 20;
    public static final double maxHumiValue = 90;

    private static double humidity = 0;
    private static double temperature = 0;

    private final ScheduledExecutorService scheduler;

    public DHT11Class() {
        this.scheduler = Executors.newSingleThreadScheduledExecutor(new NamedThreadFactory("DHT11"));
        scheduler.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    readTemperatureAndHumidity();
                } catch (Exception e) {
                    System.out.println("Error reading sensor");
                    e.printStackTrace();
                }
            }
        }, 2, 10, TimeUnit.SECONDS);
    }

    private synchronized void readTemperatureAndHumidity() throws Exception {
        // TODO Auto-generated method stub

        Runtime rt = Runtime.getRuntime();
        Process p = rt.exec("python AdafruitDHT.py 11 25");
        BufferedReader bri = new BufferedReader(new InputStreamReader(p.getInputStream()));
        if ((line = bri.readLine()) != null) {
            if (!(line.contains("ERR_CRC") || line.contains("ERR_RNG"))) {
                data = line.split(" ");
                temperature = Double.parseDouble(data[0]);
                humidity = Double.parseDouble(data[1]);
            } else {
                System.out.println("Error reading sensor value");
            }
        }
        bri.close();
        p.waitFor();

        setChanged();
        notifyObservers();
    }

    public synchronized double getTemperature() {
        return temperature;
    }

    public synchronized double getHumidity() {
        return humidity;
    }

}
```
The DHTClass represents the DHT11 hardware and supplies all the hardware characteristics like minimiun and maximium range of the sensor to the other classes.

To notify other classes when the temperature and humidity value changes the DHT11 class extends from Observable.


#### 2- Create the Humidity class to represent the 3304.xml

To represent the humidity lwm2m object we need to create the HumditySensor class:

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Observable;
import java.util.Observer;
import java.util.Random;

import org.eclipse.leshan.client.resource.BaseInstanceEnabler;
import org.eclipse.leshan.core.response.ExecuteResponse;
import org.eclipse.leshan.core.response.ReadResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HumiditySensor extends BaseInstanceEnabler implements Observer {

    private static final Logger LOG = LoggerFactory.getLogger(HumiditySensor.class);

    private static final int SENSOR_VALUE = 5700;
    private static final int MIN_MEASURED_VALUE = 5601;
    private static final int MAX_MEASURED_VALUE = 5602;
    private static final int MIN_RANGE_VALUE = 5603;
    private static final int MAX_RANGE_VALUE = 5604;
    private static final String SENSOR_UNITS = "%";
    private static final int RESET_MIN_MAX_MEASURED_VALUES = 5605;
    private static final int UNITS = 5701;

    private final Random rng = new Random();
    private double currentHumidity = 30d;
    private double minMeasuredValue = currentHumidity;
    private double maxMeasuredValue = currentHumidity;

    private DHT11Class dht11;

    public HumiditySensor(DHT11Class dht11) {
        this.dht11 = dht11;
        this.dht11.addObserver(this);
    }

    @Override
    public synchronized ReadResponse read(int resourceId) {
        switch (resourceId) {
        case SENSOR_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(currentHumidity));
        case MIN_MEASURED_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(minMeasuredValue));
        case MAX_MEASURED_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(maxMeasuredValue));
        case MIN_RANGE_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(dht11.minHumiValue));
        case MAX_RANGE_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(dht11.maxHumiValue));
        case UNITS:
            return ReadResponse.success(resourceId, SENSOR_UNITS);
        default:
            return super.read(resourceId);
        }
    }

    @Override
    public synchronized ExecuteResponse execute(int resourceId, String params) {
        switch (resourceId) {

        case RESET_MIN_MAX_MEASURED_VALUES:
            resetMinMaxMeasuredValues();
            return ExecuteResponse.success();
        default:
            return super.execute(resourceId, params);
        }
    }

    @Override
    public void update(Observable o, Object arg) {
        currentHumidity = dht11.getHumidity();

        Integer changedResource = adjustMinMaxMeasuredValue(currentHumidity);
        if (changedResource != null) {
            fireResourcesChange(SENSOR_VALUE, changedResource); // Must be responsible to generate the notification
                                                                // to observers
        } else {
            fireResourcesChange(SENSOR_VALUE);
        }
    }

    private Integer adjustMinMaxMeasuredValue(double newHumidity) {

        if (newHumidity > maxMeasuredValue) {
            maxMeasuredValue = newHumidity;
            return MAX_MEASURED_VALUE;
        } else if (newHumidity < minMeasuredValue) {
            minMeasuredValue = newHumidity;
            return MIN_MEASURED_VALUE;
        } else {
            return null;
        }
    }

    private double getTwoDigitValue(double value) {
        BigDecimal toBeTruncated = BigDecimal.valueOf(value);
        return toBeTruncated.setScale(2, RoundingMode.HALF_UP).doubleValue();
    }

    private void resetMinMaxMeasuredValues() {
        minMeasuredValue = currentHumidity;
        maxMeasuredValue = currentHumidity;
    }
}
```
The class extends the BaseInstanceEnabler class from leshan.client.resource and implments the Observer in order to receive the updates of sensor data when new data is collected.

From the BaseInstanceEnabler class the HumiditySensor overrides two methods: the read() and the execute(). 
* The read method handles the read requests made by the leshan server, returning the value of corresponding recource ID;
* The execute method receive commands from the leshian server and execute operations;

From the Observer interface, the class overrides the update method. This method is called when new data is collected by the DHT11 class. The method updates the sensor variables like: min, max and current value.

#### 3- Modify the TemperatureSensor class to support the DHT11 class

To the DHT11 class be supported in the TemperatureSensor class, the class needs to implements the Observer to receive the update messages. Thus, the constructor of the class was modified and the override method was created.

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Observable;
import java.util.Observer;
import java.util.Random;

import org.eclipse.leshan.client.resource.BaseInstanceEnabler;
import org.eclipse.leshan.core.response.ExecuteResponse;
import org.eclipse.leshan.core.response.ReadResponse;

public class TemperatureSensor extends BaseInstanceEnabler implements Observer {

    private static final String UNIT_CELSIUS = "cel";
    private static final int SENSOR_VALUE = 5700;
    private static final int UNITS = 5701;
    private static final int MAX_MEASURED_VALUE = 5602;
    private static final int MIN_MEASURED_VALUE = 5601;
    private static final int RESET_MIN_MAX_MEASURED_VALUES = 5605;

    private final Random rng = new Random();
    private double currentTemp = 20d;
    private double minMeasuredValue = currentTemp;
    private double maxMeasuredValue = currentTemp;

    private DHT11Class dht11;

    public TemperatureSensor(DHT11Class dht11) {
        this.dht11 = dht11;
        this.dht11.addObserver(this);
    }

    @Override
    public synchronized ReadResponse read(int resourceId) {
        switch (resourceId) {
        case MIN_MEASURED_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(minMeasuredValue));
        case MAX_MEASURED_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(maxMeasuredValue));
        case SENSOR_VALUE:
            return ReadResponse.success(resourceId, getTwoDigitValue(currentTemp));
        case UNITS:
            return ReadResponse.success(resourceId, UNIT_CELSIUS);
        default:
            return super.read(resourceId);
        }
    }

    @Override
    public synchronized ExecuteResponse execute(int resourceId, String params) {
        switch (resourceId) {
        case RESET_MIN_MAX_MEASURED_VALUES:
            resetMinMaxMeasuredValues();
            return ExecuteResponse.success();
        default:
            return super.execute(resourceId, params);
        }
    }

    private double getTwoDigitValue(double value) {
        BigDecimal toBeTruncated = BigDecimal.valueOf(value);
        return toBeTruncated.setScale(2, RoundingMode.HALF_UP).doubleValue();
    }

    @Override
    public void update(Observable o, Object arg) {
        currentTemp = dht11.getTemperature();
        Integer changedResource = adjustMinMaxMeasuredValue(currentTemp);
        if (changedResource != null) {
            fireResourcesChange(SENSOR_VALUE, changedResource);
        } else {
            fireResourcesChange(SENSOR_VALUE);
        }
    }

    private Integer adjustMinMaxMeasuredValue(double newTemperature) {

        if (newTemperature > maxMeasuredValue) {
            maxMeasuredValue = newTemperature;
            return MAX_MEASURED_VALUE;
        } else if (newTemperature < minMeasuredValue) {
            minMeasuredValue = newTemperature;
            return MIN_MEASURED_VALUE;
        } else {
            return null;
        }
    }

    private void resetMinMaxMeasuredValues() {
        minMeasuredValue = currentTemp;
        maxMeasuredValue = currentTemp;
    }
}
```

#### 4- Add the new class to the LeshanClientDemo

Lastly, to add the new humidity class and instance in the Leshan client demo we need to do the following steps:

4.1- Add and initialize a DHT11 object;

4.2- Add the new model to the available models;
```java
private final static String[] modelPaths = new String[] { "3303.xml", "3304.xml" };
```

4.3- Add the identifier for the new object:
```java
private static final int OBJECT_ID_HUMIDITY_SENSOR = 3304;
```

4.4- Set the instance for the new type sensor:
```java
initializer.setInstancesForObject(OBJECT_ID_HUMIDITY_SENSOR, new HumiditySensor(dht11));
```

4.5- Add the new object in the enablers list:
```java
List<LwM2mObjectEnabler> enablers = initializer.create(SECURITY, SERVER, DEVICE, LOCATION, OBJECT_ID_TEMPERATURE_SENSOR, OBJECT_ID_HUMIDITY_SENSOR); 
```

Finally, the class modified:
```java
import static org.eclipse.leshan.LwM2mId.*;
import static org.eclipse.leshan.client.object.Security.*;

import java.io.File;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.List;
import java.util.Scanner;

import org.apache.commons.cli.CommandLine;
import org.apache.commons.cli.DefaultParser;
import org.apache.commons.cli.HelpFormatter;
import org.apache.commons.cli.Options;
import org.apache.commons.cli.ParseException;
import org.eclipse.californium.core.network.config.NetworkConfig;
import org.eclipse.leshan.LwM2m;
import org.eclipse.leshan.client.californium.LeshanClient;
import org.eclipse.leshan.client.californium.LeshanClientBuilder;
import org.eclipse.leshan.client.object.Server;
import org.eclipse.leshan.client.resource.LwM2mObjectEnabler;
import org.eclipse.leshan.client.resource.ObjectsInitializer;
import org.eclipse.leshan.core.model.LwM2mModel;
import org.eclipse.leshan.core.model.ObjectLoader;
import org.eclipse.leshan.core.model.ObjectModel;
import org.eclipse.leshan.core.request.BindingMode;
import org.eclipse.leshan.util.Hex;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LeshanClientDemo {

    private static final Logger LOG = LoggerFactory.getLogger(LeshanClientDemo.class);

    private final static String[] modelPaths = new String[] { "3303.xml", "3304.xml" }; // Added

    private static final int OBJECT_ID_TEMPERATURE_SENSOR = 3303;
    private static final int OBJECT_ID_HUMIDITY_SENSOR = 3304; // Added
    private final static String DEFAULT_ENDPOINT = "LeshanClientDemo";
    private final static String USAGE = "java -jar leshan-client-demo.jar [OPTION]";

    private static DHT11Class dht11;

    private static MyLocation locationInstance;

    public static void main(final String[] args) {

        // Define options for command line tools
        Options options = new Options();

        options.addOption("h", "help", false, "Display help information.");
        options.addOption("n", true, String.format(
                "Set the endpoint name of the Client.\nDefault: the local hostname or '%s' if any.", DEFAULT_ENDPOINT));
        options.addOption("b", false, "If present use bootstrap.");
        options.addOption("lh", true, "Set the local CoAP address of the Client.\n  Default: any local address.");
        options.addOption("lp", true,
                "Set the local CoAP port of the Client.\n  Default: A valid port value is between 0 and 65535.");
        options.addOption("slh", true, "Set the secure local CoAP address of the Client.\nDefault: any local address.");
        options.addOption("slp", true,
                "Set the secure local CoAP port of the Client.\nDefault: A valid port value is between 0 and 65535.");
        options.addOption("u", true, String.format("Set the LWM2M or Bootstrap server URL.\nDefault: localhost:%d.",
                LwM2m.DEFAULT_COAP_PORT));
        options.addOption("i", true,
                "Set the LWM2M or Bootstrap server PSK identity in ascii.\nUse none secure mode if not set.");
        options.addOption("p", true,
                "Set the LWM2M or Bootstrap server Pre-Shared-Key in hexa.\nUse none secure mode if not set.");
        options.addOption(
                "pos",
                true,
                "Set the initial location (latitude, longitude) of the device to be reported by the Location object. Format: lat_float:long_float");
        options.addOption("sf", true, "Scale factor to apply when shifting position. Default is 1.0.");
        HelpFormatter formatter = new HelpFormatter();
        formatter.setOptionComparator(null);

        // Parse arguments
        CommandLine cl;
        try {
            cl = new DefaultParser().parse(options, args);
        } catch (ParseException e) {
            System.err.println("Parsing failed.  Reason: " + e.getMessage());
            formatter.printHelp(USAGE, options);
            return;
        }

        // Print help
        if (cl.hasOption("help")) {
            formatter.printHelp(USAGE, options);
            return;
        }

        // Abort if unexpected options
        if (cl.getArgs().length > 0) {
            System.err.println("Unexpected option or arguments : " + cl.getArgList());
            formatter.printHelp(USAGE, options);
            return;
        }

        // Abort if we have not identity and key for psk.
        if ((cl.hasOption("i") && !cl.hasOption("p")) || !cl.hasOption("i") && cl.hasOption("p")) {
            System.err.println("You should precise identity and Pre-Shared-Key if you want to connect in PSK");
            formatter.printHelp(USAGE, options);
            return;
        }

        // Get endpoint name
        String endpoint;
        if (cl.hasOption("n")) {
            endpoint = cl.getOptionValue("n");
        } else {
            try {
                endpoint = InetAddress.getLocalHost().getHostName();
            } catch (UnknownHostException e) {
                endpoint = DEFAULT_ENDPOINT;
            }
        }

        // Get server URI
        String serverURI;
        if (cl.hasOption("u")) {
            if (cl.hasOption("i"))
                serverURI = "coaps://" + cl.getOptionValue("u");
            else
                serverURI = "coap://" + cl.getOptionValue("u");
        } else {
            if (cl.hasOption("i"))
                serverURI = "coaps://localhost:" + LwM2m.DEFAULT_COAP_SECURE_PORT;
            else
                serverURI = "coap://localhost:" + LwM2m.DEFAULT_COAP_PORT;
        }

        // get security info
        byte[] pskIdentity = null;
        byte[] pskKey = null;
        if (cl.hasOption("i") && cl.hasOption("p")) {
            pskIdentity = cl.getOptionValue("i").getBytes();
            pskKey = Hex.decodeHex(cl.getOptionValue("p").toCharArray());
        }

        // get local address
        String localAddress = null;
        int localPort = 0;
        if (cl.hasOption("lh")) {
            localAddress = cl.getOptionValue("lh");
        }
        if (cl.hasOption("lp")) {
            localPort = Integer.parseInt(cl.getOptionValue("lp"));
        }

        // get secure local address
        String secureLocalAddress = null;
        int secureLocalPort = 0;
        if (cl.hasOption("slh")) {
            secureLocalAddress = cl.getOptionValue("slh");
        }
        if (cl.hasOption("slp")) {
            secureLocalPort = Integer.parseInt(cl.getOptionValue("slp"));
        }

        Float latitude = null;
        Float longitude = null;
        Float scaleFactor = 1.0f;
        // get initial Location
        if (cl.hasOption("pos")) {
            try {
                String pos = cl.getOptionValue("pos");
                int colon = pos.indexOf(':');
                if (colon == -1 || colon == 0 || colon == pos.length() - 1) {
                    System.err.println("Position must be a set of two floats separated by a colon, e.g. 48.131:11.459");
                    formatter.printHelp(USAGE, options);
                    return;
                }
                latitude = Float.valueOf(pos.substring(0, colon));
                longitude = Float.valueOf(pos.substring(colon + 1));
            } catch (NumberFormatException e) {
                System.err.println("Position must be a set of two floats separated by a colon, e.g. 48.131:11.459");
                formatter.printHelp(USAGE, options);
                return;
            }
        }
        if (cl.hasOption("sf")) {
            try {
                scaleFactor = Float.valueOf(cl.getOptionValue("sf"));
            } catch (NumberFormatException e) {
                System.err.println("Scale factor must be a float, e.g. 1.0 or 0.01");
                formatter.printHelp(USAGE, options);
                return;
            }
        }

        createAndStartClient(endpoint, localAddress, localPort, secureLocalAddress, secureLocalPort, cl.hasOption("b"),
                serverURI, pskIdentity, pskKey, latitude, longitude, scaleFactor);
    }

    public static void createAndStartClient(String endpoint, String localAddress, int localPort,
            String secureLocalAddress, int secureLocalPort, boolean needBootstrap, String serverURI,
            byte[] pskIdentity, byte[] pskKey, Float latitude, Float longitude, float scaleFactor) {

        locationInstance = new MyLocation(latitude, longitude, scaleFactor);
        dht11 = new DHT11Class();

        // Initialize model
        List<ObjectModel> models = ObjectLoader.loadDefault();
        models.addAll(ObjectLoader.loadDdfResources("/models", modelPaths));

        // Initialize object list
        ObjectsInitializer initializer = new ObjectsInitializer(new LwM2mModel(models));
        if (needBootstrap) {
            if (pskIdentity == null)
                initializer.setInstancesForObject(SECURITY, noSecBootstap(serverURI));
            else
                initializer.setInstancesForObject(SECURITY, pskBootstrap(serverURI, pskIdentity, pskKey));
        } else {
            if (pskIdentity == null) {
                initializer.setInstancesForObject(SECURITY, noSec(serverURI, 123));
                initializer.setInstancesForObject(SERVER, new Server(123, 30, BindingMode.U, false));
            } else {
                initializer.setInstancesForObject(SECURITY, psk(serverURI, 123, pskIdentity, pskKey));
                initializer.setInstancesForObject(SERVER, new Server(123, 30, BindingMode.U, false));
            }
        }
        initializer.setClassForObject(DEVICE, MyDevice.class);
        initializer.setInstancesForObject(LOCATION, locationInstance);
        initializer.setInstancesForObject(OBJECT_ID_TEMPERATURE_SENSOR, new TemperatureSensor(dht11));
        initializer.setInstancesForObject(OBJECT_ID_HUMIDITY_SENSOR, new HumiditySensor(dht11)); // Added
        List<LwM2mObjectEnabler> enablers = initializer.create(SECURITY, SERVER, DEVICE, LOCATION,
                OBJECT_ID_TEMPERATURE_SENSOR, OBJECT_ID_HUMIDITY_SENSOR); // Added

        // Create CoAP Config
        NetworkConfig coapConfig;
        File configFile = new File(NetworkConfig.DEFAULT_FILE_NAME);
        if (configFile.isFile()) {
            coapConfig = new NetworkConfig();
            coapConfig.load(configFile);
        } else {
            coapConfig = LeshanClientBuilder.createDefaultNetworkConfig();
            coapConfig.store(configFile);
        }

        // Create client
        LeshanClientBuilder builder = new LeshanClientBuilder(endpoint);
        builder.setLocalAddress(localAddress, localPort);
        builder.setLocalSecureAddress(secureLocalAddress, secureLocalPort);
        builder.setObjects(enablers);
        builder.setCoapConfig(coapConfig);
        // if we don't use bootstrap, client will always use the same unique endpoint
        // so we can disable the other one.
        if (!needBootstrap) {
            if (pskIdentity == null)
                builder.disableSecuredEndpoint();
            else
                builder.disableUnsecuredEndpoint();
        }
        final LeshanClient client = builder.build();

        LOG.info("Press 'w','a','s','d' to change reported Location ({},{}).", locationInstance.getLatitude(),
                locationInstance.getLongitude());

        // Start the client
        client.start();

        // De-register on shutdown and stop client.
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                client.destroy(true); // send de-registration request before destroy
            }
        });

        // Change the location through the Console
        try (Scanner scanner = new Scanner(System.in)) {
            while (scanner.hasNext()) {
                String nextMove = scanner.next();
                locationInstance.moveLocation(nextMove);
            }
        }
    }
}
```

All the executables jar files and the python application are availble in [files folder](http://gitlabsocialite.dei.uc.pt/Socialite/lwm2m-tutorial/tree/master/files).

Video with the [client](https://www.youtube.com/watch?v=9CPdKtIUC0U) running...
