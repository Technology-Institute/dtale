Hello everyone.  My name is Andrew Schonfeld and wanted to tell you a little bit about
my experiences writing Flask applications, specifically my open-source package, D-Tale.

I've been a software engineer in the finance industry for over 14 years in Boston.  The first 8 writing Java spring applications for big banks and then next 7 developing Python Flask applications for a small quant shop.  More recently, in the midst of a pandemic, I moved to Pittsburgh to start a new job at Argo AI (a software company specializing in autonomous vehicles) mainly working on Typescript applications.

During my time working for that small quant shop in Boston, Numeric Investors, it was where I found out how to build web applications with Python using a little package called Flask.  It was a revelation compared to the work I had done in Java.  With just a few lines of code I had a fully functioning web app.  So off I went initially building full-scale standalone web applications sourcing data from SQL Server and Mongo to jinja & requirejs fueled front-ends.  A few years later I found out about the magic of ReactJS and started building my frontends using that.  A little while after that I worked with my CI/CD department to find a way to package my Flask apps as deployable eggs and later more sustainably using Docker.  The one constant throughout this process was Flask.

Eventually, that small quant shop was purchased by one of the largest international hedge funds, MAN Group, and it was under their eye that I was recommended to start open-sourcing my projects.  The first project I was tasked with finding an easy way to visualize pandas dataframes.  Our company had recently completed a conversion from SAS to Python and needed some way to replicate the visualizer they had for their SAS datasets.  One of the first requirements was that it had to be able to be called on-demand from a library when users were debugging code or working in a jupyter notebook.  This was something I had never done before using Flask because all of my applications were long-running processes.

So after some thinking I came up with the idea of kicking off a flask process within a separate thread from the main process they were working in:

```python
import _thread
from flask import Flask

DATA = None
def _start(df):
    global DATA

    DATA = df

    app = Flask(__name__)

    @app.route('/row-count')
    def row_count():
        return 'Dataframe has {} rows'.format(len(DATA))

    app.run(host='0.0.0.0', port=8080, debug=False, threaded=True)

_thread.start_new_thread(_start, ())
```

This effectively spawns a Flask process on port 8088 of the localhost that can allow users to continue working in the main process and share memory with the Flask process (via the `DATA` variable).

So originally the idea was to allow users to spin up a Flask process for each piece of data they wanted to view.  Essentially each dataframe would occupy a different port on the user's machine.  This quickly became unwieldy and so I decided to go a different route allow multiple pieces of data to be viewed from one Flask process.  I was able to accomplish this by converting the `DATA` variable from one dataframe into a dictionary of multiple dataframes.  The keys of dictionary would be identifiers that can be used as variables within the routes of our Flask process.

```python

DATA = {
    '1': df1,
    '2': df2
}

@app.route('/row-count/<data_id>')
def row_count(data_id=None):
    df = DATA.get(data_id)
    return 'Dataframe {} has {} rows'.format(data_id, len(df))
```

So to view the row count for dataframe 1 use the URL `http://localhost:8080/row-count/1`.
Recently I just added a feature where users can also assign names to their data which can be used within their URLs to retrieve their data.

```
NAMES = {
    'foo': '1',
    'bar': '2'
}

@app.url_value_preprocessor
def handle_data_id(_endpoint, values):
    if values and "data_id" in values and values["data_id"] not in DATA:
        values["data_id"] = NAMES[values["data_id"]]
```

Now that URL becomes `http://localhost:8080/row-count/foo`.

Another feature which was uncharted territory for me was the ability to have some sort of "reaper" for these Flask processes that users opened and forget about.  So the way I was able to solve this issue was by initializing a `Timer` object which after an hour will fire a request to a "shutdown" (which is another process to tackle) route on your Flask process.

```python
import requests
import sys
from threading import Timer
from flask import Flask, request

class MyFlask(Flask):

    def __init__(self, import_name, *args, **kwargs):
        self.start_reaper()
        super(MyFlask, self).__init__(import_name, *args, **kwargs)
    
    def build_reaper():
        def _func():
            # make sure the Flask process is still running
            if is_up('http://localhost:{}'.format(self.port)):
                requests.get(self.shutdown_url)
            sys.exit()  # kill off the reaper thread

        self.reaper = Timer(60.0 * 60.0, _func)  # one hour
        self.reaper.start()
    
    def clear_reaper(self):
        if self.reaper:
            self.reaper.cancel()
    
    def run(self, *args, **kwargs):
        self.port = str(kwargs.get("port") or "")
        self.build_reaper()
        super(MyFlask, self).run(*args, **kwargs)


def is_up(base):
    try:
        return requests.get("{}/health".format(base), verify=False).ok
    except BaseException:
        return False

app = MyFlask(__name__)

@app.route("/health")
def health_check():
    return "ok"

@app.before_request
def before_request():
    # before each request we want to re-initialize the reaper since we only
    # want to kill the process after inactivity
    app.build_reaper()

@app.route("/shutdown")
def shutdown():
    app.clear_reaper()
    func = request.environ.get("werkzeug.server.shutdown")
    if func is None:
        raise RuntimeError("Not running with the Werkzeug Server")
    func()
    return "Server shutting down..."
```

