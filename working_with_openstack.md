# Working with OpenStack

**Author**: [Lee Napthine](/team/lee-napthine)  
**Last updated**: 2025-03-25

**Originally published at:** [https://arcsoft.uvic.ca/log/2025-01-31-working-in-openstack/](https://arcsoft.uvic.ca/log/2025-01-31-working-in-openstack/)

Recently, I built a program that directly interacts with OpenStack cloud operations and its
various components. It was a solid learning experience, and I want to share what I learned so
future ARCsoft developers can hit the ground running.

## Cloud Infrastructure

Just in case - let’s quickly define cloud. A cloud is a network of remote servers
that store, manage, and process data over the internet or a private network instead of on local
machines. This is particularly useful for scaling projects that may have differing size demands.

[Google Cloud](https://cloud.google.com/discover/types-of-cloud-computing) offers some
definitions on the different types of clouds (summarized):

- **Public Cloud** – Provided by third-party companies (e.g., AWS, Google Cloud, Azure).
- **Private Cloud** – Hosted by an organization for internal use (e.g., Company
  VMs typically run on a private cloud server).
- **Hybrid Cloud** – A combination of public and private cloud environments.

I would offer a fourth type, which we use here at the University of Victoria Research Computing
Services:

- **Community Cloud** – A cloud that is provided for a specific community (e.g.,
  OpenStack at Arbutus which is offered to the research community)

Cloud computing offers benefits such as scalability, cost-efficiency, and remote accessibility.
OpenStack is an example of a private cloud framework that allows organizations to build and
manage their own clouds.

## OpenStack

OpenStack is an open-source cloud computing platform that provides both an interface and a
customizable cloud infrastructure. It enables users to manage large pools of compute, storage,
and networking resources, similar to AWS or Google Cloud but in a self-hosted or private cloud
environment.

Everything we do at ARCsoft is connected to an OpenStack cloud. Our virtual machines are hosted
in the cloud, the projects that are outsourced to us are developed in the cloud, and all other
operations at the Research Computing Services are conducted in the cloud. The sooner we have an
understanding of what it is and its many mechanics, the better we will be at interfacing with
the backend infrastructure we use daily.

[Arbutus](https://www.uvic.ca/systems/researchcomputing/about-us/arbutus/index.php) is
Canada’s largest research cloud. Users interact with Arbutus primarily through the
OpenStack [Horizon dashboard](https://docs.OpenStack.org/horizon/latest/), accessible
at <https://arbutus.cloud.computecanada.ca>. This web interface allows users to manage and monitor cloud resources, including launching and managing virtual machines (VMs), configuring networks, and setting up storage resources.

Let’s dive into the Horizon dashboard:

Openstack’s Horizon dashboard is their web-based user interface to services including Nova,
Swift, Keystone, etc. (more on these later). It allows control of their cloud platform without
needing to interact with the command line. Here, users can manage projects, hypervisors, and
adjust the administrative settings based on their roles and permissions.

Usually, users will navigate to Horizon by visiting: **https://<name_of_cloud>/dashboard**.

Once logged in, the dashboard is organized into several tabs:

- **Project Tab** - For users to manage their cloud resources, API access, and network settings.
- **Admin Tab** - For administrators to oversee and manage the entire cloud infrastructure.
- **Identity Tab** - To manage projects, users, and roles.

Horizon is a great place to start. Try playing around here. You will develop a better
understanding of the component parts of the OpenStack cloud going forward.

## Components

Let’s briefly summarize the most important components that we will work with while
developing on OpenStack:

- **Nova**: This is the central compute component. When you are working with the
  virtual machines (instances) on OpenStack, you are using Nova. Most operations directly deal
  with Nova; scheduling, scaling, launching, and terminating “instances”, for
  example.
- **Neutron**: OpenStack’s networking service. Neutron manages IP address
  allocation, network topology, and security groups. I played around with it in Horizon, but
  my experience with it is limited.
- **Cinder**: Cinder deals with block storage: block-allocatable persistent
  storage. This storage underlies filesystems (as opposed to “object storage”).
- **Keystone**: Keystone handles authentications and authorizations. Anything
  related to users, roles, and projects will interact with Keystone. It is OpenStack’s
  central identity service.

There are other components, but I used these the most and I believe that others will too.

## Command-Line Interface

By learning OpenStack’s command-line interface (CLI), we can access and manage cloud
resources more quickly than through Horizon. We can even use it from within custom scripts and
programs for more customized control.

Let’s go over what we will need in order to access the OpenStack CLI.

**Step one** - From the Horizon dashboard, select the API access tab, and from the
top-right-corner drop down box labeled ‘Download OpenStack RC file’, select the
export for the OpenStack `clouds.yaml` file.

**Step two** - Store the `clouds.yaml` file at the path
`~/.config/openstack/clouds.yaml`. Follow the guidelines listed in the commented code
provided directly in the `.yaml` file. You’ll want to set any information
specific to your roles and projects. Change the `openstack:` line to match the name
of your cloud access. In my case, it was `devstack.lee:`.

**Step three** - You will likely want to be working from within a virtual
environment prior to installing anything. Set the environment up by inputting
`python -m venv venv` into your command line and activate it using:
`. venv/bin/activate`.

**Step four** - Install the OpenStack client with:

```
pip install openstackclient
```

**Step five** - Install the OpenStack SDK with:

```
pip install openstacksdk
```

**Step six** - From your terminal, enter

```
OS_COMPUTE_API_VERSION=2.92 OS_CLOUD=devstack.lee openstack
```

(replace the name of your cloud and the version you want to use).

You’re in!

Let’s go over some of what we will see when using the OpenStack CLI.

Typing `server`, or `project` will show you the list of commands accessible
on these objects. The same goes for all objects in OpenStack cloud. For an extensive list visit
[here](https://docs.openstack.org/python-openstackclient/pike/cli/command-list.html).

Here is a small example:

```
server list
```

This will give you a short description of all servers on the cloud. Grab a server ID and enter:

```
 server show <server_id>
```

This will get you a much more detailed description of a particular instance.

Play around in the command line client - try tagging and adding metadata properties!

This leads us to the project we were tasked with developing.

## Scanner and Action

We designed our own command-line interface for scanning through the cloud looking at individual
projects and instances that may have violated certain conditions. For example, if resources
allocated to a researcher have exceeded the allowable lifetime, our program will identify the
violation, notify everyone involved, and potentially take actions shelving or deleting the
resources. For a full description of the project, visit [this
document](https://gitlab.com/uvic-arcsoft/calm/cip/-/blob/main/docs/plan.md?ref_type=heads) in our repository.

Instead of relying on the OpenStack command-line interface (CLI) to interact with cloud
resources, we chose to integrate our program directly with OpenStack’s API. This approach
provides greater flexibility, automation, and scalability, allowing us to scan, tag, and take
action on cloud resources directly.

## API

Let’s explain why we use the API:

1. **Automation** – Accessing APIs allow us to write scripts and programs that run
   independently.
2. **Integration** – The API can communicate directly with OpenStack services.
3. **Customization** – We can filter, sort, and analyze cloud resources in any
   format we want rather than relying on CLI outputs.

To import the Openstack API into your program:

```
import openstack
```

In our case, we needed a way to interact with cloud instances while also handling
OpenStack-specific exceptions to identify errors during operations.

```
from openstack.compute.v2.server import Server
from openstack.exceptions import ConflictException,OtherSpecifiedExceptions,Etc...
```

Next, establish a connection to the OpenStack cloud to create a reference
(`self._conn`). This allows you to interact with cloud resources and execute API operations.

```python
def __init__(self, config):
    # Create connection to OpenStack with specified API version
    self._conn = openstack.connect(cloud=config.cloud, compute_api_version='2.92')
    if self._conn is None:
        raise ConnectionError("Failed to establish a connection to the OpenStack cloud.")
```

By integrating the OpenStack API, we built a system that lets us manage cloud resources directly
from the terminal. Automating these operations made our workflow more efficient and gave us more
control than using the CLI alone.

## Conclusion

If you’re working with OpenStack, the key takeaway is this: learn the API. The CLI is great
for quick tasks, but real power comes from automation. Understanding OpenStack’s
architecture—Nova for compute, Neutron for networking, Cinder for storage, and Keystone for
authentication—will give you a huge advantage when developing cloud-based tools.

Now that you’ve got the basics, play around, break things, and figure out what works best. The
world is yours. See what you can come up with!

---
