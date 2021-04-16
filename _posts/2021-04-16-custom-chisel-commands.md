---
layout: post
title:  "Implementing Chisel custom commands"
date:   2021-04-16 03:00:14 +0100
categories: chisel
---
Performing a network call that receives a JSON is one of the most common tasks that iOS developers perform nowadays. And every time that I am either developing a network interface or debugging a problem, I need to inspect what is coming from in the payload before parsing it.

Some time ago I saw a [solution by Sam Soffes](https://soffes.blog/debugging-json-data-in-lldb) where he created a LLDB command to transform the data into a dictionary and print the content.

I really liked it, but since I am also a big fan of [chisel](https://github.com/facebook/chisel), I took the opportunity to learn how to write a custom command for it. So in his post, I will write about how to load custom chisel commands and how to print JSON content of a data blob.

## Loading custom commands
It is pretty easy to load your custom commands. After you decide where you will store them (I chose ~/.chisel), you need to load the directory in your `.lldbinit` file. Mine looks like this:
```
command script import /usr/local/opt/chisel/libexec/fbchisellldb.py
script fbchisellldb.loadCommandsInDirectory('/Users/dbarden/.chisel')
```


## Skeleton of a custom command

The skeleton of a custom command looks like this:

{% highlight python linenos %}
#!/usr/bin/python

import lldb
import fbchisellldbbase as fb

def lldbcommands():
  return [ CustomCommand() ]
  
class CustomCommand(fb.FBCommand):
    def name(self):
        return "customCommand"

    def description(self):
        return "Sample implementation of a custom command"

    def args(self):
        return [
            fb.FBCommandArgument(
                arg="object", 
                type="id", 
                help="Object expression to be evaluated."
            )
        ]

    def options(self):
        return [
            fb.FBCommandArgument(
                arg="appleWay",
                short="-a",
                long="--apple",
                boolean=True,
                default=False,
                help="Print ivars the apple way",
            )
        ]

    def run(self, arguments, options):
		
{% endhighlight %}

Lines 6-7 expose the instance of the class that will be used by LLDB to run the commands.

Then it comes the command definition. I assume that the methods are pretty much self explanatory. Note that not all of them are mandatory.

## JSON Command
To pretty print the data, I wanted to run the following code:

{% highlight swift %}
func json(input: Any) {
  let object: Any

  if let data = input as? Data {
    object = try! JSONSerialization.jsonObject(with: input, options: []))
  } else {
    return
  }

  // Convert the object pretty printed JSON data
  let jsonData = try! JSONSerialization.data(withJSONObject: object, options: [.prettyPrinted])

  // Convert the pretty printed JSON data to a `String`
  let jsonString = String(data: jsonData, encoding: .utf8)!

  // Print the result
  print(jsonString)
}

{% endhighlight %}

To implement this as command in chisel, let's first implement the data about name, description and arguments:

{% highlight python %}
#!/usr/bin/python

import lldb
import fbchisellldbbase as fb

def lldbcommands():
  return [ JSON() ]
  
class JSON(fb.FBCommand):
  def name(self):
    return 'json'
    
  def description(self):
    return 'Print the json representation from a data object.'
    
  def args(self):
    return [
      fb.FBCommandArgument(
        arg="data",
        type="NSData",
        help="Data that will be evaluated when deserializing.",
      )
    ]
{% endhighlight %}

The `run` command should be able to parse the data into an object and transform into a structure (either a dictionary or array) and be serialised again to be pretty printed. The snippet 

{% highlight python%}
  def run(self, arguments, options):
    # It's a good habit to explicitly cast the type of all return
    # values and arguments. LLDB can't always find them on its own.
    inputData = fb.evaluateInputExpression("{obj} as NSData".format(obj=arguments[0]))

    jsonData = fb.evaluateExpression("[NSJSONSerialization JSONObjectWithData:(NSData *){data} options:nil error:nil]".format(data=inputData))
    objectData = fb.evaluateExpression("[NSJSONSerialization dataWithJSONObject:(NSObject *){data} options:NSJSONWritingPrettyPrinted error:nil]".format(data=jsonData))
    string = fb.describeObject("(NSString*)[[NSString alloc] initWithData:(NSData *){data} encoding:4]".format(data=objectData))
{% endhighlight %}

To execute the commands, we run the `fb.evaluateExpression` method. This method will output the hex value of the return variable.

There are other variants, like `evaluateIntegerExpression` or `evaluateBooleanExpression`, to be used with other native types.

To use this output as a parameter in the next command, we need to explicitly cast it into the expected type, since it's just a hex value.

When we want to describe the object, we can call the `fb.describeObject` method. This should be enough to print the JSON Payload. Let's take a look at how to use it.

## Using the command
My preferred way to use commands like this is to create a breakpoint and run the command from there. This way, we don't need to add any code to debug the payload:
![](/assets/json_breakpoint.png)

## Conclusion
This post introduced how to create a custom command in chisel and implemented a way to print the JSON payload of a `Data` instance. These steps can be used to extend chisel in any other common scripting that you use LLDB for.