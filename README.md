# OMG Monitor

This program uses [NuPIC] to catch anomalies in [Pingdom] monitored response times. It runs as a [Docker] container, so to use it you just have to run the container (see [Usage](#usage)).

Here is a simplified flowchart of the project:

![flowchart](https://rawgithub.com/cloudwalkio/omg-monitor/images/images/omg-monitor-flow.svg)

## Input

The input are response times gathered by [Pingdom]. We use the [python-restful-pingdom] module to get the Pingdom data used to train the [NuPIC] model. The model is first trained with the last 5000 response times of the specified checks,
after which it starts learning online, making a request per minute per check to Pingdom. 

## Output

The calculated anomaly score for each input (plus some information) is stored in a [Redis] server. The data stored in Redis are lists of strings of the form `"time,status,actual,predicted,anomaly,likelihood"` with keys of the form `"results:[CHECK_ID]"`.

## API

The results can be accessed via a RESTful API written in [Go] using [Martini].

It is very simplistic:

* To get the servers available in the supplied Pingdom account:
```
/checks
```
The JSON string returned contains only the IDs and names of the checks.

* To get the last `[N]` results for `[CHECK_ID]`:
```
/results/[CHECK_ID]?limit=[N]
```
  The resulting JSON string has the following fields:
  * actual: Actual response time at the given time instant.
  * predicted: Response time prediction for the given time instant.
  * anomaly: the unlikelihood of the actual result in comparisson with the predicted result.
  * status: status of the server as returned by Pingdom (up, down, unconfirmed_down).
  * time: UNIX time when Pingdom got the actual response time.
  If no limit is specified it is assumed that `N=0`, so that the API returns all the results for the given `CHECK_ID`.

## API Client

The Go server also serves static HTML files that uses [jQuery] to access our API to get the results and dinamically plot them. Currently we have three visualizations:

* The [index.html][1] file uses [justGage] to plot the latest anomalies likelihoods. 
* The [anomaly.html][2] file uses [D3.js] to plot the last hour results with anomalies. 
* The [likelihood.html][3] file uses [D3.js] to plot the last hour results with anomalies likelihoods. 

See the session [Screenshots](#screenshots) for some examples.

## Usage

With [Docker] installed, do:
```
sudo docker run allanino/monitor -d -p [PUBLIC_PORT]:5000 [USERNAME] [PASSWORD] [APPKEY] [CHECK_ID_1] [CHECK_ID_2] ...
```

The parameters that we must specify:

* `[PUBLIC_PORT]`: Public port number used by the Go server.

* `[USERNAME] [PASSWORD] [APPKEY]`: Pingdom's credentials: username, password and app-key.

* (Optional) `[CHECK_ID_1] [CHECK_ID_2] ...`: Pingdom's IDs to monitor. If we don't specify any IDs, the monitor will use all available IDs from Pingdom.

## Q&A

### What happens when the container is started?

The Docker container entrypoint is the script [startup.sh], responsible for starting the Redis and Go servers and for running the [start.py] script, which will start one [monitor.py] instance for each check, each one in a separate thread (running in parallel). The script [monitor.py] contains the NuPIC code. The [start.py] script will also fetch the available checks from the given Pingdom account and save them in the Redis server with key `"checks"` and value as a list of strings with the IDs. It will also create keys of the form `"check:[CHECK_ID]"` that stores the corresponding check's name. All that information is used by our API. 

As soon as the Go server is available, we can see the logs generated by Redis, Martini and the monitors in the URL `/log`. 

### How to build the image?

The above Docker command `run` will pull the image `allanino/monitor` from [Docker index][docker_image]. That image is kept updated through Docker's Trusted Build feature.

To build the Docker image locally, clone this repository and do:

    sudo docker build -t "[USERNAME]/monitor" .

Note that our [Dockerfile] uses the [allanino/nupic] image, as that image already contains a NuPIC installation.

## Screenshots


![gauge](https://rawgithub.com/cloudwalkio/omg-monitor/images/images/gauge.png)

![anomaly](https://rawgithub.com/cloudwalkio/omg-monitor/images/images/anomaly.png)

![likelihood](https://rawgithub.com/cloudwalkio/omg-monitor/images/images/likelihood.png)

License
-------
```
OMG Monitor
Copyright (C) 2014 Cloudwalk

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

```

[NuPIC]:https://github.com/numenta/nupic
[Docker]:https://www.docker.io/
[Pingdom]:https://www.pingdom.com/
[Redis]:http://redis.io/
[Martini]:https://github.com/codegangsta/martini
[Go]:http://golang.org/
[D3.js]:http://d3js.org/
[jQuery]:http://jquery.com/
[justGage]:http://justgage.com/
[python-restful-pingdom]:https://github.com/drcraig/python-restful-pingdom
[allanino/nupic]:https://github.com/allanino/docker-nupic

[Dockerfile]:https://github.com/allanino/omg-monitor/blob/master/Dockerfile
[monitor.py]:https://github.com/allanino/omg-monitor/blob/master/monitor/monitor.py
[start.py]:https://github.com/allanino/omg-monitor/blob/master/start.py
[startup.sh]:https://github.com/allanino/omg-monitor/blob/master/startup.sh
[docker_image]:https://index.docker.io/u/allanino/monitor/
[1]:https://github.com/allanino/omg-monitor/blob/master/server/public/index.html
[2]:https://github.com/allanino/omg-monitor/blob/master/server/public/anomaly.html
[3]:https://github.com/allanino/omg-monitor/blob/master/server/public/likelihood.html

