===================================
OpenStack Instance Deployment Guide
===================================

This guide details the standard procedure for deploying virtual machine (VM)
instances through the project dashboard. The workflow is split into three
sections based on the availability zone of the instance:

* :ref:`compute-deployment` - standard CPU workloads.
* :ref:`gpu-deployment` - GPU / AI workloads.
* :ref:`radio-deployment` - wireless / SDR workloads (includes USRP booking).

Every user follows the same path: read :ref:`before-you-start`, log in, create
your network and router **once**, create a :ref:`volume <volume-management>`
to hold your data, then launch whichever instance type you need. Finish with
the :ref:`common post-launch steps <common-post-launch>` and connect to your
instance through :ref:`terminal-access`.

.. contents::
   :local:
   :depth: 2


.. _before-you-start:

Before You Start
=================

Read this section before launching anything - these are the points that most
often trip up new users.

* **Create your network first**: A brand-new project has no usable network.
  You must create your own network and router before any instance can be
  launched (see :ref:`network-setup`). You only do this once per project.
* **Choose the right availability zone**: ``compute`` for CPU workloads,
  ``gpu`` for GPU/AI workloads, and ``radio`` for SDR/wireless workloads.
* **Watch your quotas**: Instance, VCPU, RAM, and volume limits are shown on
  the **Overview** tab. New instances will fail to create once a quota is hit.
* **Terminal access is over ZeroTier**: There is no public SSH and no floating
  IP. You reach every instance - compute, GPU, or radio - over the project's
  ZeroTier network (see :ref:`terminal-access`).
* **GPU flavors need approval**: Request the GPU flavor through the booking
  portal before attempting a GPU launch, or the launch will fail.
* **Book USRPs first**: A radio VM cannot reach an SDR unit that has not been
  reserved for the matching time slot.
* **Persistent vs ephemeral storage**: Set **Create New Volume = Yes** to keep
  data across rebuilds; ephemeral disks are wiped when the instance is deleted.
  Without a volume, data **cannot be retrieved** if anything happens to the
  server. Take regular volume snapshots and push your work to your own
  GitHub/GitLab repository (see :ref:`volume-management`).
* **Change default credentials**: Update the default image password
  (``CCI@2025``) on first login for any production workload.
* **Free up resources**: Delete unused instances and volumes to stay within
  quota and free them for other project members.


.. _onboarding:

Onboarding & Logging In
=======================

New users do **not** receive login credentials by email. Access is granted
through CILogon / authentik once your account is approved.

* **Log in**: Open the OpenStack application from authentik. In the login
  dropdown, choose **Authenticate with authentik**, then select the **domain**
  named in your welcome email.
* **Resource Quotas**: The **Overview** tab displays the project's limits.
  For example, a project may be restricted to 10 instances, 20 VCPUs, and
  50 GB of RAM.
* **Quota Management**: Once these limits are reached, the system prevents the
  creation of new instances until you free resources.

.. note::

   Default **image credentials** for all Ubuntu-based images in this guide are
   ``Username: ubuntu`` / ``Password: CCI@2025``. These log you into the
   instance itself - they are not OpenStack credentials and not SSH keys.
   Change the password on first login for production workloads.


.. _network-setup:

Set Up Your Network and Router
==============================

A new project has no private network of its own. Before launching any
instance, create one network and one router that bridges it to the provider
network. **You only need to do this once per project** - the same network and
router are reused by every instance you launch.

Create a Network
----------------

Navigate to **Network → Networks** and click **+ Create Network**. The wizard
has three tabs:

* **Network** tab:

  * **Network Name**: ``external-network-<firstname>`` (for example,
    ``external-network-JohnDoe``).
  * Leave **Enable Admin State** and **Create Subnet** checked.
  * **MTU**: set to ``1450``.

* **Subnet** tab:

  * **Subnet Name**: ``external-subnet-<firstname>``.
  * **Network Address Source**: keep **Enter Network Address manually**.
  * **Network Address**: a private CIDR, for example ``10.120.0.0/24``.
  * Leave **IP Version** as ``IPv4`` and leave **Gateway IP** blank to accept
    the default (the first address in the range). Do not tick
    **Disable Gateway**.

* **Subnet Details** tab: enable **DHCP**, then click **Create**.

Create a Router
---------------

Navigate to **Network → Routers** and click **+ Create Router**.

* **Router Name**: give it a name (for example,
  ``Internal-Project-<Firstname>-Router``).
* Leave **Enable Admin State** checked.
* **External Network**: select ``Main-Internet-Network``.
* Click **Create Router**.

Add an Interface
----------------

Connect your subnet to the router so your private network can reach the
external network:

1. From **Network → Routers**, click your newly created router to open it.
2. Go to the **Interfaces** tab and click **+ Add Interface**.
3. In the **Subnet** dropdown, select ``external-subnet-<firstname>``.
4. Leave **IP Address (optional)** blank.
5. Click **Submit**.

The new internal interface appears in the list with a fixed IP and a status of
``Active``.

Verify the Topology
-------------------

Open **Network → Network Topology** to confirm the layout: your router should
be linked to ``Main-Internet-Network`` on one side and to your
``external-subnet-<firstname>`` on the other. Radio users will also see the
USRP networks here once an interface is attached.

.. tip::

   Once the network and router exist, they are reused for every instance you
   launch in the project - you only create them once.


.. _volume-management:

Volume Management
=================

Volumes let you boot instances from persistent storage and keep data across
rebuilds. Create your volume **before** launching an instance, then select it
as the boot **Source** in the launch procedure.

.. warning::

   If you launch an instance **without a volume**, its data lives only on the
   server's local disk. If anything happens to the server, that data
   **cannot be retrieved**. To protect your work:

   * Boot from a **volume** (or set **Create New Volume = Yes**) for every
     instance you care about.
   * Take a **volume snapshot** regularly (see :ref:`volume-snapshot`), so you
     can restore your volume if something goes wrong.
   * Push your code and important data to **your own GitHub or GitLab
     repository** for version control and an off-testbed backup.

Creating a Volume
-----------------

1. Navigate to the left-hand sidebar and select **Volumes**, then click on
   **Volumes** from the dropdown.
2. Click the **+ Create Volume** button located near the top right of the
   dashboard.
3. Provide a name for the volume in the designated field.
4. Select your **Volume Source**. To build from a backup, select
   **Volume Snapshot** and choose the specific snapshot from the dropdown.
5. Specify the desired capacity in the **Size (GiB)** field (e.g., ``128``).
6. Click the blue **Create Volume** button to initialize the storage.


.. _volume-snapshot:

Creating a Volume Snapshot
--------------------------

1. From the **Volumes** list, locate the volume you want to back up.
2. Click the dropdown arrow in the **Actions** column and select
   **Create Snapshot**.
3. Enter a snapshot name (for example, include the date).
4. Click **Create Volume Snapshot**.

The snapshot appears under **Volumes → Snapshots**. To restore from it, create
a new volume with **Volume Snapshot** as the source (see
**Creating a Volume** above).

Extending a Volume (Increasing Size)
------------------------------------

1. From the **Volumes** list, locate the specific volume you want to
   resize.
2. Click the dropdown arrow next to **Edit Volume** in the **Actions**
   column.
3. Select **Extend Volume** from the menu.
4. Enter the new, larger capacity in the **New Size (GiB)** field (for
   example, upgrading from ``32`` to ``128``).
5. Click **Extend Volume** to apply the new size limit.


Attaching a Volume (Connecting to an Instance)
----------------------------------------------

1. From the **Volumes** list, locate the volume you want to connect.
2. Click the dropdown arrow in the **Actions** column for that volume.
3. Select **Manage Attachments**.
4. Choose the target instance from the dropdown menu where you want the
   volume mounted.
5. Click **Attach Volume** to link the storage to your virtual machine.


.. _compute-deployment:

Compute Instance Deployment
===========================

.. raw:: html

   <div class="demo-videos">
     <h3>How to Create a Compute Node</h3>
     <video controls preload="metadata" playsinline crossorigin="anonymous" width="640">
       <source src="../_static/Compute.mp4" type="video/mp4">
       Your browser does not support the video tag.
     </video>
     <p><a href="../_static/Compute.mp4">Download video</a></p>
   </div>

Launch Procedure
----------------

* Navigate to **Project → Compute → Instances** and click **Launch Instance**.
* **Details**: Enter an Instance Name (e.g., ``test_compute``).
  Set the Availability Zone to ``compute``.
* **Source**:

  * Set **Select Boot Source** to **Image** (or **Volume**, if you are booting
    from a volume you created - see :ref:`volume-management`).
  * Set **Create New Volume** to **No** (ephemeral, wiped on delete) for
    standard testing, or **Yes** (persistent, saved to Cinder storage) to keep
    critical data.
  * Select your desired image or volume and click the **Up Arrow (↑)** to
    move it into **Allocated**.

* **Flavor**: Choose a flavor (e.g., ``m4.4.32``) and click the
  **Up Arrow (↑)**.
* **Networks**: Allocate your own network, ``external-network-<firstname>``.
* **Security Groups**: Allocate the ``default`` security group.
* **Launch**: Click **Launch Instance** and wait for the status to change to
  ``Active`` and power state to ``Running``.

Once the instance is ``Active``, connect to it over ZeroTier (see
:ref:`terminal-access` at the end of this guide), then see
:ref:`common-post-launch` for storage and security steps.

.. tip::

   Monitor the instance row through the sequence
   ``Build`` → ``Spawning`` → ``Active``. If it stalls on ``Build`` for more
   than a few minutes, check your project quotas on the Overview tab.


.. _gpu-deployment:

GPU Instance Deployment
=======================

.. raw:: html

   <div class="demo-videos">
     <h3>How to Create and Configure a GPU Node</h3>
     <video controls preload="metadata" playsinline crossorigin="anonymous" width="640">
       <source src="../_static/GPU-instance.mp4" type="video/mp4">
       Your browser does not support the video tag.
     </video>
     <p><a href="../_static/GPU-instance.mp4">Download video</a></p>
   </div>

.. important::

   Before creating a GPU instance, you must request a GPU flavor, specifying
   the GPU version you need, from the **USRP & GPU Access** tab of the booking
   portal (see :ref:`portal-tabs`). Without an approved flavor, the launch
   step below will fail.

Launch Procedure
----------------

* Navigate to **Project → Compute → Instances** and click **Launch Instance**.
* **Details**: Enter an Instance Name (e.g., ``test_gpu``).
  Set the Availability Zone to ``gpu``.
* **Source**: Choose **Image** from the dropdown menu (or a **Volume** you
  created). Select your source and click the **Up Arrow (↑)**.

  .. note::

     Use **Persistent** storage (Create New Volume = Yes) if you plan to
     retain large datasets across instance rebuilds.

* **Flavor**: Choose your approved GPU flavor and click the **Up Arrow (↑)**.
* **Networks**: Allocate your own network, ``external-network-<firstname>``.
* **Security Groups**: Allocate the ``default`` security group.
* **Launch**: Click **Launch Instance** and wait for the status to change to
  ``Active``.

Once the instance is ``Active``, connect to it over ZeroTier (see
:ref:`terminal-access` at the end of this guide), then continue with the GPU
configuration below.


GPU Configuration & Validation
------------------------------

Install Drivers
~~~~~~~~~~~~~~~

Install the kernel headers, build tools, and NVIDIA driver stack
(version 535):

.. code-block:: bash

   sudo apt update
   sudo apt install -y linux-headers-$(uname -r) \
                       linux-modules-extra-$(uname -r) \
                       build-essential dkms ubuntu-drivers-common
   sudo apt install -y nvidia-driver-535 nvidia-dkms-535 \
                       nvidia-utils-535 libnvidia-compute-535
   sudo dkms autoinstall -k $(uname -r)
   sudo update-initramfs -u
   sudo reboot

.. important::

   The instance **must** reboot after driver installation. Reconnect over
   ZeroTier once it returns to ``Active``.


Verify Drivers
~~~~~~~~~~~~~~

After reconnecting, confirm the GPU is detected:

.. code-block:: bash

   nvidia-smi

Expected output: a table listing the GPU model (e.g., Tesla V100), driver
version 535.x, and CUDA version.


Install PyTorch and Validate CUDA
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   sudo apt update
   sudo apt install -y python3-pip
   python3 -m pip install --upgrade pip numpy
   python3 -m pip install torch --index-url https://download.pytorch.org/whl/cu121

Quick sanity check:

.. code-block:: bash

   python3 - <<'PY'
   import torch
   print("CUDA available:", torch.cuda.is_available())
   print("GPU name:", torch.cuda.get_device_name(0))
   PY


GPU Stress Test
~~~~~~~~~~~~~~~

Execute a 5-minute matrix multiplication loop to verify hardware stability:

.. code-block:: bash

   python3 - <<'PY'
   import torch, time
   assert torch.cuda.is_available(), "CUDA is not available"
   device = "cuda"
   print("GPU:", torch.cuda.get_device_name(0))
   size = 12000
   a = torch.randn(size, size, device=device)
   b = torch.randn(size, size, device=device)
   end = time.time() + 300
   count = 0
   while time.time() < end:
       c = torch.matmul(a, b)
       torch.cuda.synchronize()
       count += 1
   print("Completed matrix multiplications:", count)
   PY

In a second terminal, monitor utilization and check for hardware errors:

.. code-block:: bash

   watch -n 1 nvidia-smi
   sudo dmesg | grep -i "xid"

.. note::

   A successful test returns **no output** from the ``xid`` grep, meaning
   zero Xid errors were detected. GPU utilization should stay between
   90-100 % during the run.


.. _radio-deployment:

Radio Instance Deployment
=========================

.. raw:: html

   <div class="demo-videos">
     <h3>How to Create a Radio Node</h3>
     <video controls preload="metadata" playsinline crossorigin="anonymous" width="640">
       <source src="../_static/Radio-Instance.mp4" type="video/mp4">
       Your browser does not support the video tag.
     </video>
     <p><a href="../_static/Radio-Instance.mp4">Download video</a></p>
   </div>

Deploying a radio instance is a **two-stage process**:

1. **Book a USRP** on the CCI xG Testbed booking portal (see below).
2. **Launch a Radio VM** in the ``radio`` availability zone to act as the
   host for the booked USRP.

.. important::

   You must complete the USRP booking **before** launching the radio VM.
   The VM will not be able to communicate with an SDR unit you have not
   reserved.


Booking USRP Resources (SDR Booking)
------------------------------------

Reserve - and cancel - USRP nodes from the booking portal. The portal uses a
tabbed layout: **Book USRP**, **My Reservations**, **USRP & GPU Access**,
**OpenStack Quota**, and **Resources**.

.. raw:: html

   <div class="demo-videos">
     <h3>How to Book a USRP</h3>
     <video controls preload="metadata" playsinline crossorigin="anonymous" width="640">
       <source src="../_static/USRP-booking-new.mp4" type="video/mp4">
       Your browser does not support the video tag.
     </video>
     <p><a href="../_static/USRP-booking-new.mp4">Download video</a></p>
   </div>

Prerequisites
~~~~~~~~~~~~~

* An approved CCI xG account and your OpenStack project name
  (e.g., ``External-Project-<firstname>``).

Open the Booking Portal
~~~~~~~~~~~~~~~~~~~~~~~

Sign in to the CCI xG dashboard at https://authentik.ccixgtestbed.org/ and open
the booking portal, then select the **Book USRP** tab (or click **Start
booking** on the landing page).

Validate Your OpenStack Project
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A pop-up prompts you to **Enter Your OpenStack Project**. Type your project
name, click **Validate**, then **Continue**.

.. note::

   The project name is case-sensitive and must match your OpenStack project
   exactly. If validation fails, double-check it against your OpenStack
   dashboard, where it appears in the project selector at the top left,
   next to the OpenStack logo (shown as ``<domain> • <project>``).

Select Date Range, Mode, and USRP
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On the **Book USRP** page, under **Select Date Range and Time**:

* **USRP Selection**: choose **Single USRP** or **Multiple USRPs**.
* Pick a **From Date** and **To Date**.
* Bookings are issued in fixed **4-hour time blocks**: 12 AM-4 AM, 4 AM-8 AM,
  8 AM-12 PM, 12 PM-4 PM, 4 PM-8 PM, and 8 PM-12 AM (Eastern Time).
* Your **project limit** is shown alongside the controls (e.g.,
  ``1 of 3 USRPs currently reserved``).

Scroll to **Available USRPs**. Nodes are arranged by their physical row/radio
position, each tile showing the node number, model (e.g., ``X310`` or
``N310``), and a **Book** action. Tiles are colour-coded:

* **Green** - Available.
* **Yellow** - Partially Available (some slots in the range are taken).
* **Red** - Fully Booked.
* **Grey** - Disabled or under Maintenance.

Click **Book** on an available node to open its reservation dialog.

Configure the Cellular Plan and Confirm
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The **Book USRP <id>** dialog opens a **Cellular plan** form. Only FCC-safe,
bookable bands are shown.

* **Radio standard**: e.g., ``5G NR`` or ``LTE 4G``.
* **Duplex mode**: ``TDD`` or ``FDD``.
* **Band**: e.g., ``NR n78 (TDD)``.
* **Bandwidth (MHz)**: up to ``20`` MHz (the SDR-based UE maximum).
* **Carrier input**: choose **Downlink center frequency** or **Downlink
  ARFCN**, then enter the value.

The backend calculates the uplink/downlink carrier ranges from the band,
duplex mode, and bandwidth, enforces a 5 MHz guard band between users, and
shows any occupied slices and free windows for the band.

* Tick **I agree to the Acceptable Use Policy**.
* Click **Review Booking** to see the summary (project, USRP, dates, time,
  radio plan, bandwidth, and computed downlink/uplink ranges).
* Click **Confirm Booking**. A confirmation email is sent immediately.

.. warning::

   Operating outside the approved cellular plan, or changing radio settings
   without approval, may result in a ban from using testbed devices. See the
   Acceptable Use Policy for details.

.. tip::

   Need a longer experiment? Book consecutive 4-hour blocks on the same node.
   For several radios at once, use **Multiple USRPs** in the selection step.

Managing Reservations (View, Cancel, Change)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Open the **My Reservations** tab to manage active bookings. Each row lists the
Booking ID, USRP, project, band, start/end time, status, and **Port**. Ports
are created automatically when the reservation starts; until then the Port
column shows **Pending** with an **Attach guide** link. Once active, attach the
port to your instance following the SDR network configuration steps below.

* **Cancel**: tick the reservation's checkbox, then click **Cancel selected**
  (only possible before the reservation starts).
* **Request a change**: click the calendar action icon to open **Request
  Reservation Change**. Set **Request type** (e.g., ``Modify time``), enter the
  **Proposed start**/**Proposed end** and a **Reason**, then **Submit
  Request**.


Launch Procedure (Radio VM)
---------------------------

Once your USRP is booked, launch the radio host instance:

* Navigate to **Project → Compute → Instances** and click **Launch Instance**.
* **Details**: Enter an Instance Name. Set the Availability Zone to
  ``radio``.
* **Source**: Choose **Image** from the dropdown menu (or a **Volume** you
  created). Select your source and click the **Up Arrow (↑)**.
* **Flavor**: Choose an appropriate radio flavor and click the
  **Up Arrow (↑)**.
* **Networks**: Allocate your own network, ``external-network-<firstname>``.
* **Security Groups**: Allocate the ``default`` security group.
* **Launch**: Click **Launch Instance** and wait for the status to change
  to ``Active``.

Once the instance is ``Active``, connect to it over ZeroTier (see
:ref:`terminal-access` at the end of this guide), then continue with the SDR
network configuration below.


SDR Network Configuration
-------------------------

Because the USRP nodes exist on an isolated private network, you must manually
attach the radio network to your instance and configure the internal routing
before you can interact with the SDR. Connect to the instance over
:ref:`ZeroTier <terminal-access>` first.

Step A: Attach the USRP Interface
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. In the OpenStack dashboard, navigate to **Project → Compute → Instances**.
2. Click the dropdown arrow next to your Radio instance and select
   **Attach Interface**.
3. Change the "Way to specify an interface" dropdown to **by Port**.
4. Select the specific USRP port you booked (e.g., ``USRP-110``) from the
   dropdown list.
5. Click **Attach Interface**.
6. Check your Instances list. Your VM should now display two IP addresses
   (e.g., your network IP and your USRP Network IP, such as
   ``192.168.110.7``). Take note of this USRP IP address.

Step B: Configure Netplan
~~~~~~~~~~~~~~~~~~~~~~~~~~

Connected over ZeroTier, configure Ubuntu to recognize the newly attached
radio interface (usually ``ens4``).

1. Run ``ip a`` to verify the new interface name. You should see ``ens4``
   listed without an IP address.
2. Find the netplan configuration file (on cloud images it is often
   ``50-cloud-init.yaml``), then open it, replacing ``<file-name>`` with the
   name listed:

   .. code-block:: bash

      ls /etc/netplan/
      sudo nano /etc/netplan/<file-name>.yaml

3. Add your ``ens4`` interface and the USRP IP address you noted from the
   OpenStack dashboard (append ``/24`` to it). The file should look like this:

   .. code-block:: yaml

      network:
        ethernets:
          ens3:
            dhcp4: true
          ens4:
            addresses: [192.168.110.7/24]
        version: 2

4. Save and exit the file (Ctrl+O, Enter, Ctrl+X).
5. Apply the new network configuration:

   .. code-block:: bash

      sudo netplan apply

Step C: SDR Verification
~~~~~~~~~~~~~~~~~~~~~~~~~

Once the network is applied, verify connectivity to the USRP unit. The SDR is
typically located at the ``.2`` address of your subnet (e.g., if your IP is
``192.168.110.7``, the radio is at ``192.168.110.2``) - confirm the exact
address for your booked node.

.. code-block:: bash

   ping 192.168.110.2     # verify reachability to the radio node
   uhd_find_devices       # discover the booked USRP node

.. note::

   ``uhd_find_devices`` should list the USRP unit you reserved in the
   :ref:`booking section <radio-deployment>`. If it returns nothing, confirm
   your booking is still active in the **My Reservations** tab of the booking
   portal, and verify that your netplan configuration has the correct IP
   address.


.. _portal-tabs:

Booking Portal: Other Tabs
==========================

Besides booking USRPs, the portal at https://authentik.ccixgtestbed.org/
exposes three more tabs for requesting capacity and checking status.

USRP & GPU Access
-----------------

Use this tab to request additional radio or GPU capacity for your project
beyond your current allocation. Fill in the request form and submit it; your
submissions and their status appear under **My USRP Requests** / **My GPU
Requests**. This is also where you request the GPU flavor referenced in
:ref:`GPU Instance Deployment <gpu-deployment>`.

OpenStack Quota
---------------

Use this tab to request increases to your OpenStack compute quota (instances,
VCPUs, RAM, volumes). The form shows recommended values and deployment
guidance, and your prior requests appear in the request history. Your current
limits are visible on the OpenStack **Overview** tab (see
:ref:`Before You Start <before-you-start>`).

Resources
---------

Use this tab to check live testbed status: GPU availability, maintenance
windows (including any that overlap your selected booking range), operational
notices, and links to the deployment guides.


.. _common-post-launch:

Common Post-Launch Procedures
=============================

The following procedures apply to **all** instance types (compute, GPU, and
radio) once the VM is ``Active``.


Expanding Instance Storage
--------------------------

If you launched with a disk size larger than the default Instance Snapshot,
you must manually expand the storage:

.. code-block:: bash

   sudo apt update
   sudo apt install -y cloud-guest-utils lvm2
   sudo growpart /dev/vda 3
   sudo pvresize /dev/vda3
   sudo lvextend -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv


.. _network-connectivity:

Network Connectivity & Security
-------------------------------

All terminal access is over ZeroTier (see :ref:`terminal-access`), so the
``default`` security group only needs the rules that let ZeroTier and your
workload through.

* **ZeroTier transport**: Navigate to **Network → Security Groups** and click
  **Manage Rules** for the ``default`` group. Add the following:

  * **Custom UDP Rule** - Port ``9993`` (ZeroTier's default transport port).
  * **All ICMP** - to allow ``ping`` for reachability checks across the
    ZeroTier network.

* **SSH over ZeroTier**: Add an **SSH** (TCP ``22``) rule, but restrict the
  CIDR to the project's ZeroTier subnet rather than ``0.0.0.0/0`` so the port
  is not exposed to the public internet.
* **Additional protocols**: Add any application-specific ports your experiment
  needs (e.g., HTTP/HTTPS, custom TCP/UDP ranges).

.. warning::

   Never open SSH to ``0.0.0.0/0``. There is no public access path in this
   testbed - keep all access scoped to the ZeroTier overlay.


.. _terminal-access:

Terminal Access (ZeroTier)
==========================

There is no public SSH access and no floating IP in this testbed. You reach
**every** instance - compute, GPU, and radio - over the project's **ZeroTier**
overlay network.

All testbed images (Ubuntu 18.04, 20.04, 22.04, and 24.04) come with ZeroTier
already installed, so on each instance you only need to join the project
network and get it authorized. On your local machine, install ZeroTier from
https://www.zerotier.com/download/, then join and authorize it the same way.

.. note::

   The project's **16-digit ZeroTier network ID** and device authorization are
   managed by the CCI admin team. Request the network ID and have your devices
   authorized before you begin.

Join the Project Network
------------------------

.. code-block:: bash

   sudo zerotier-cli join <network-id>

Replace ``<network-id>`` with the 16-digit hexadecimal ID from the admin team.

Authorize the Device
--------------------

Print this device's 10-digit ID and send it to the admin team so they can
authorize it on the ZeroTier controller:

.. code-block:: bash

   sudo zerotier-cli info

Until the device is authorized, the network shows ``ACCESS_DENIED``.

Confirm Your Assigned IP
------------------------

Once authorized, list your networks and note the assigned ZeroTier IP:

.. code-block:: bash

   sudo zerotier-cli listnetworks

The network status changes to ``OK`` and an IP address is listed.

Connect
-------

From your authorized local machine, SSH to the instance's ZeroTier IP using the
image credentials (``ubuntu`` / ``CCI@2025``):

.. code-block:: bash

   ssh ubuntu@<instance-zerotier-ip>
