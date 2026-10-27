.. _soloran-guide:

=============
SoloRAN Guide
=============

SoloRAN takes an approved CCI xG Testbed researcher from account sign-in
through deployment of a RAN VM and a Near-RT RIC VM, component integration,
traffic generation, Python xApp execution, monitoring, recovery, and cleanup.
This guide uses the prepared ``transfer-ran`` and ``transfer-ric-srsran`` images.

* **SoloRAN application:** https://soloran.ccixgtestbed.org
* **OpenStack dashboard:** https://portal.ccixgtestbed.org
* **Source guide version:** 1.0, 25 August 2026.

For general project, network, quota, and instance concepts, read the
:doc:`OpenStack Guide <openstack_v3>` first.

.. important::

   This SoloRAN workflow uses floating IPs so the **SoloRAN server** can reach
   the prepared VMs over SSH. The general OpenStack Guide describes
   **ZeroTier** access for users. Before deploying, ask the testbed
   administrator to confirm that your project supports the SoloRAN management
   path, floating-IP pool, and security-group rules described here.

.. contents::
   :local:
   :depth: 2


Video Tutorial
==============

Watch the SoloRAN user walkthrough alongside the step-by-step instructions
below.

.. raw:: html

   <div class="demo-videos">
     <h3>Using SoloRAN on the CCI xG Testbed</h3>
     <video controls preload="metadata" playsinline crossorigin="anonymous" width="800">
       <source src="../_static/SoloRAN-user-guide.mp4" type="video/mp4">
       Your browser does not support the video tag.
     </video>
     <p><a href="../_static/SoloRAN-user-guide.mp4">Download video</a></p>
   </div>


Workflow at a Glance
====================

Authenticate → deploy RIC → deploy RAN → associate floating IPs → enter
runtime addresses → start Core, RIC, gNB, and UE → validate traffic → run
an xApp → observe → stop or recover → clean up.

SoloRAN provisions cloud resources, controls the runtime lifecycle, generates
traffic, and executes Python xApps against a personal or approved shared
Near-RT RIC.

.. list-table:: Resources and responsibilities
   :header-rows: 1
   :widths: 18 25 57

   * - Resource
     - Prepared image
     - What runs there
   * - RAN VM
     - ``transfer-ran``
     - Open5GS Core, O-CUDU gNB, srsUE, UE namespace, and ping/iPerf client.
   * - RIC VM
     - ``transfer-ric-srsran``
     - Dockerized O-RAN SC Near-RT RIC and ``python_xapp_runner``.
   * - SoloRAN server
     - Managed service
     - Identity/session handling, OpenStack API access, deployment, SSH
       orchestration, logs, and charts.

.. important::

   Wait for the success gate in each terminal before clicking the next
   control. A button click starts an asynchronous operation. Repeated clicks
   can create duplicate stacks or overlapping startup tasks.


Before You Start
================

Access and Quotas
-----------------

Confirm that you have:

* An approved authentik identity with access to SoloRAN.
* A correctly mapped OpenStack project with enough compute, RAM, disk,
  port, and floating-IP quota for two instances.
* Visibility of the ``transfer-ran`` and ``transfer-ric-srsran`` images in
  your project.
* A tenant network and security group that let the SoloRAN management path
  reach SSH/TCP port ``22`` on each VM.
* Internal RAN-to-RIC connectivity, including SCTP port ``36421`` for E2.
  Ask the testbed administrator to confirm project security policy; avoid
  unnecessarily broad public ingress.

Know Your Account Path
----------------------

.. list-table:: Account paths
   :header-rows: 1
   :widths: 15 25 40 20

   * - Account type
     - SoloRAN sign-in
     - OpenStack connection
     - Typical project
   * - Lion
     - **Continue with Authentik**
     - Established through federated sign-in; no second OpenStack password.
     - ``External-Project-JohnDoe``
   * - Hummingbird
     - Authenticate identity, then enter Keystone credentials.
     - Keystone password validated for the current server-side session.
     - ``Internal-Project-Maheshav``

.. important::

   Open the profile menu immediately after sign-in. Confirm that the
   displayed OpenStack project belongs to you. If it is missing or
   unexpected, sign out and contact the administrator before deploying.


Authenticate to SoloRAN
=======================

Lion User
---------

1. Open https://soloran.ccixgtestbed.org.
2. Click **Lion: Continue with Authentik** and complete the authentik prompt.
3. If **Connect OpenStack with Authentik** appears, click it once and allow
   the redirect to finish.
4. Verify that the profile menu shows your verified email, an
   ``External-Project-*`` project, and an active OpenStack connection.

Hummingbird User
----------------

1. Complete the authentik identity step used to enter SoloRAN.
2. On the Hummingbird form, verify the read-only username and enter your
   OpenStack/Keystone password.
3. Click **Continue to dashboard**. The password is retained only in
   encrypted server-side session memory.
4. Verify that the profile menu shows your ``Internal-Project-*`` project
   and an active OpenStack connection.

.. note::

   If OpenStack calls begin returning ``401`` or resource lists become
   unavailable, use **Profile → Manage session**. Use **Sign out** when
   finished, especially on a shared workstation. Never include credentials
   in screenshots or support logs.


Deploy the Near-RT RIC Instance
===============================

1. Choose **Near RT RIC Deployment** in the left navigation.
2. Select **My Personal Project** for a new personal RIC. The shared RIC
   project is reserved for approved shared experiments; normal users cannot
   deploy into it.
3. Enter a short, unique instance name, such as ``alice-ric``, and record it
   in the experiment worksheet below.
4. Select image ``transfer-ric-srsran``, an allowed availability zone, a
   flavor that satisfies the image requirements, the project network, and
   the project key name if required by policy. Keep visibility **Private**
   unless your experiment explicitly requires otherwise.
5. Click **DEPLOY INSTANCE** once. Follow the **Deployment Terminal** while
   Heat creates the stack.
6. Continue only when the terminal reports ``CREATE_COMPLETE`` /
   ``Deployment Finished Successfully`` and the instance is **Active** in
   OpenStack.

.. note::

   The first runtime start on a new RIC may take longer while SoloRAN seeds
   the verified Docker image cache and initializes Redis, E2 Manager,
   E2Term, and Subscription Manager. Do not click **Start RIC** repeatedly.


Deploy the RAN Instance
=======================

1. Choose **RAN Deployment** in the left navigation.
2. Enter a different unique name, such as ``alice-ran``.
3. Select image ``transfer-ran``, the testbed-approved availability zone
   and flavor, the same tenant network used by the RIC, and an allowed key
   name.
4. Click **DEPLOY INSTANCE** once and monitor the Heat output.
5. Continue only after ``CREATE_COMPLETE`` /
   ``Deployment Finished Successfully`` and **Active** status in OpenStack.

The prepared ``transfer-ran`` VM normally hosts the Open5GS Core, O-CUDU gNB,
and srsUE together. Therefore, **Core VM IP** and **RAN/gNB/UE VM IP** usually
point to the same floating IP.


Associate Floating IPs in OpenStack
===================================

Each instance receives a fixed tenant IP, typically ``10.0.0.x``. SoloRAN
uses a floating IP to reach the VM over SSH, while the runtime automatically
resolves fixed addresses for internal E2 traffic.

1. Open https://portal.ccixgtestbed.org and select the same project shown
   in your SoloRAN profile menu.
2. Go to **Project → Compute → Instances**.
3. On the RIC row, open **Actions → Associate Floating IP**.
4. Select an unused address. If none is available, use the **+** control,
   select the administrator-provided external pool, and allocate an address
   to the project.
5. Select the port whose fixed IP belongs to the RIC instance, then click
   **Associate**.
6. Repeat for the RAN instance, using a different floating IP and its
   corresponding port.
7. Refresh **Instances**. Confirm that each row shows both its fixed and
   floating address, and record the mappings.

.. list-table:: Address worksheet
   :header-rows: 1
   :widths: 25 25 25 25

   * - Component
     - Instance name
     - Fixed IP
     - Floating IP
   * - RIC
     - Your RIC name
     - Assigned RIC fixed IP
     - Assigned RIC floating IP
   * - RAN/Core/gNB/UE
     - Your RAN name
     - Assigned RAN fixed IP
     - Assigned RAN floating IP

.. tip::

   If **Associate Floating IP** is unavailable, confirm that the instance
   is not in ``ERROR``, floating-IP quota and an external pool are
   available, and you selected the correct project. Do not associate an
   address with an unrelated port. If the problem persists, send instance
   and port IDs, never passwords, to the administrator.


Enter the Runtime Addresses
===========================

Return to **SoloRAN → RAN Deployment → RAN Runtime Control**.

.. list-table:: Runtime fields
   :header-rows: 1
   :widths: 25 30 45

   * - Field
     - Recommended value
     - Reason
   * - **Core VM IP**
     - RAN floating IP
     - Open5GS runs on the prepared RAN VM.
   * - **RAN/gNB/UE VM IP**
     - RAN floating IP
     - SSH target for O-CUDU, srsUE, the namespace, and traffic.
   * - **Near-RT RIC VM IP**
     - RIC floating IP
     - SSH target for the Dockerized RIC.
   * - **gNB Bind IP**
     - Leave blank unless instructed.
     - SoloRAN selects the guest's local fixed address on the route to the RIC.
   * - Directories/configuration
     - Keep the supplied defaults.
     - They match ``transfer-ran`` and ``transfer-ric-srsran``.
   * - **UE Namespace**
     - ``ue1``
     - Default isolated UE dataplane namespace.
   * - **Ping Target**
     - ``10.45.1.1`` or ``auto``
     - Open5GS UE tunnel gateway.

.. tip::

   For an immediate SSH timeout or unauthorized-target error, verify the
   floating-IP mapping and project in **Profile → My Deployments**. Then
   check that the project security group permits SSH from the SoloRAN
   management network.


.. _soloran-readiness:

Start and Integrate the Components
==================================

Use the manual sequence for a new deployment. Move forward only after the
readiness evidence below appears. **Start Sequence** uses timers and is best
reserved for a configuration already validated manually.

.. list-table:: Startup order and readiness gates
   :header-rows: 1
   :widths: 8 25 67

   * - Order
     - Control
     - Wait for this evidence
   * - 1
     - **Start Core**
     - ``Open5GS Core is ready.``
   * - 2
     - **Start RIC**
     - ``Near-RT RIC is ready.`` and
       ``E2 endpoint: this VM:36421/sctp``
   * - 3
     - **Start gNB**
     - ``N2: Connection to AMF ... completed``;
       ``E2AP: Connection to Near-RT-RIC completed``;
       ``==== gNB started ====``
   * - 4
     - **Start UE**
     - ``PDU Session Establishment successful. IP: 10.45.1.x``
   * - 5
     - **Start Dataplane Ping**
     - Replies from the Open5GS gateway. Stop ping after validation.
   * - 6
     - **Start iPerf**
     - Sender/receiver bitrate summary and exit code ``0``.

1. Click **Start Core** once. Installation messages are normal on first
   use; wait for the Core readiness line.
2. Click **Start RIC** once and wait for readiness. Use **Check E2** if you
   want a status snapshot before starting the gNB.
3. Click **Start gNB**. The dashboard resolves the RIC floating IP to its
   fixed tenant address. Confirm both AMF/N2 and RIC/E2 connections.
4. Click **Start UE**. Wait for the PDU session and assigned ``10.45.1.x``
   address before generating traffic.
5. Validate the dataplane with a short ping, then stop ping before a
   controlled iPerf experiment.


Generate Controlled iPerf Traffic
=================================

.. list-table:: iPerf settings
   :header-rows: 1
   :widths: 25 30 45

   * - Setting
     - Available choices
     - Guidance
   * - UDP bandwidth
     - 1, 2, 5, 10, or 30 Mbit/s
     - Begin at 1 or 2 Mbit/s. Rates above 10 Mbit/s can be unstable.
   * - Duration
     - 30, 60, 120, 300, or 600 seconds
     - Use 30–60 seconds for validation and longer runs for approved
       measurements.
   * - 30 Mbit/s profile
     - Stress/DoS
     - Use only in an authorized, isolated experiment. The UI requires
       confirmation.

1. Choose bandwidth and duration. Record both with the experiment timestamp.
2. Click **Start iPerf**. SoloRAN starts the server inside Open5GS and the
   UDP client inside the UE namespace.
3. Observe the **Dataplane / Cleanup** terminal for sender and receiver
   rates. Some loss and a lower receive rate are possible at high load.
4. If necessary, use **Stop iPerf**. **Clear** does not stop traffic.


Sample Experiment with PRB-Allocator
====================================

Prerequisites
-------------

* The RIC terminal reports ``Near-RT RIC is ready.``
* The gNB terminal reports an E2AP connection to the Near-RT RIC.
* The UE is attached if the xApp needs UE-level measurements.
* iPerf or another controlled workload is running when non-zero throughput
  values are expected.

Run and Observe
---------------

1. Open **xApp Deployment** and set **RIC Host IP** to the RIC floating IP.
2. Select the publicly available **PRB-Allocator** xApp.
3. Click **Run xApp** once. The runner uploads the file, installs declared
   or detected dependencies into a persistent directory for that xApp,
   resets only the Python runner, and starts Python unbuffered.
4. Read **xApp Logs and Metrics** for stdout, stderr, Python tracebacks,
   subscription messages, and printed measurements.
5. Under **Live E2 Metric**, select a supported KPM measurement. The x-axis
   uses local time, the y-axis follows the metric unit, and the latest
   120 samples are retained.
6. Click **Stop xApp** and wait for the stopped state/terminal clear before
   running another xApp on the same RIC.


Upload and Run a Python xApp
============================

Prerequisites
-------------

Use the same RIC, E2, UE, and traffic readiness checks as the PRB-Allocator
experiment above.

Upload
------

1. In **xApp Deployment**, choose **+ Upload New**.
2. Upload exactly one ``.py`` file. Enter a clear name and description.
3. Choose visibility: **Private** is visible only to you; **Public** is
   visible to other users, but only the original uploader can delete it.
4. Click **Upload Python xApp**, then select the new xApp card.

Run and Observe
---------------

1. Set **RIC Host IP** to the **RIC floating IP**, not the RAN IP.
2. Confirm the filename in **Selected Python xApp**.
3. Click **Run xApp** once. The runner uploads the file, installs declared
   or detected dependencies into its persistent directory, resets only the
   Python runner, and starts Python unbuffered.
4. Read **xApp Logs and Metrics** for output, exceptions, subscriptions,
   and measurements.
5. Select a supported measurement under **Live E2 Metric**. The chart uses
   local time and retains the latest 120 samples.
6. Click **Stop xApp** and wait for the stopped state/terminal clear before
   switching xApps on the same RIC.

.. important::

   The minimal ``python_xapp_runner`` uses fixed REST and RMR ports. Run
   **one uploaded Python xApp at a time per RIC**. Separate users should
   use their own project RIC unless a shared experiment is explicitly
   coordinated.


Write an xApp Whose Logs Appear in SoloRAN
==========================================

Logging Contract
----------------

* Text written to stdout or stderr appears in the xApp terminal. Use
  ``print(..., flush=True)`` for immediate feedback.
* Exceptions and tracebacks also appear. Do not suppress them while
  developing.
* To feed **Live E2 Metric**, print KPM lines in the form
  ``Metric: <name>, Value: [<number>]``.
* Supported chart examples include ``DRB.UEThpUl``, ``DRB.UEThpDl``,
  ``RRU.PrbUsedUl``, ``RRU.PrbUsedDl``, ``RRU.PrbAvailUl``, and
  ``RRU.PrbAvailDl``.
* A custom decision line such as ``CONTROL: allocated_prb=70%`` appears in
  the terminal even when it is not a chart metric.
* Never print passwords, tokens, private keys, or other users' data.

Minimal E2SM-KPM Monitor
------------------------

The following single-file template subscribes to uplink and downlink
throughput. Change ``METRICS`` only to measurements supported by the
connected E2 node. It uses the ``lib.xAppBase`` SDK supplied by the prepared
RIC runtime.

.. code-block:: python

   #!/usr/bin/env python3
   import argparse
   import signal
   from lib.xAppBase import xAppBase

   METRICS = ["DRB.UEThpUl", "DRB.UEThpDl"]

   class SoloRANMonitor(xAppBase):
       def __init__(self, config, http_port, rmr_port):
           super(SoloRANMonitor, self).__init__(config, http_port, rmr_port)

       def on_indication(self, agent, subscription, header, message):
           hdr = self.e2sm_kpm.extract_hdr_info(header)
           data = self.e2sm_kpm.extract_meas_data(message)
           print("RIC Indication: {}".format(hdr.get("colletStartTime")),
                 flush=True)
           for metric_name, value in data.get("measData", {}).items():
               # This exact format is recognized by the Live E2 Metric chart.
               print("Metric: {}, Value: {}".format(metric_name, value),
                     flush=True)

       @xAppBase.start_function
       def start(self, e2_node_id):
           callback = lambda agent, sub, hdr, msg: self.on_indication(
               agent, sub, hdr, msg
           )
           print("Subscribing to {} for {}".format(e2_node_id, METRICS),
                 flush=True)
           self.e2sm_kpm.subscribe_report_service_style_1(
               e2_node_id, 1000, METRICS, 1000, callback
           )

   if __name__ == "__main__":
       parser = argparse.ArgumentParser()
       parser.add_argument("--config", default="")
       parser.add_argument("--http_server_port", type=int, default=8092)
       parser.add_argument("--rmr_port", type=int, default=4562)
       parser.add_argument("--e2_node_id",
                           default="gnbd_001_001_00019b_0")
       args = parser.parse_args()

       app = SoloRANMonitor(args.config,
                            args.http_server_port,
                            args.rmr_port)
       signal.signal(signal.SIGINT, app.signal_handler)
       signal.signal(signal.SIGTERM, app.signal_handler)
       signal.signal(signal.SIGQUIT, app.signal_handler)
       app.start(args.e2_node_id)

.. note::

   The default E2 node ID, ``gnbd_001_001_00019b_0``, matches the prepared
   demonstration configuration. If the gNB node identity changes, update
   ``--e2_node_id`` or the default in the uploaded file.

Third-party Python Dependencies
-------------------------------

SoloRAN automatically detects common imports such as NumPy, pandas, SciPy,
Matplotlib, requests, PyYAML, scikit-learn, Pillow, seaborn, joblib, and plotext.
For another package, declare it near the top of the same uploaded file:

.. code-block:: python

   # SoloRAN-Requires: package-name>=1.2 another-package

Dependencies are installed in an isolated directory for that xApp on the
RIC host and reused after the Python runner or RIC containers are recreated.
Use only trusted, pinned dependencies approved for the testbed; replace the
illustrative package names and version constraint above with approved values.


If the xApp Shows No Values
===========================

.. list-table:: xApp and runtime troubleshooting
   :header-rows: 1
   :widths: 25 30 45

   * - What you see
     - Meaning
     - Action
   * - Indications with ``Value: [0.0]``
     - Subscription works, but there is no measurable traffic.
     - Start controlled iPerf traffic and keep the UE attached.
   * - ``503 Service Unavailable``
     - RIC Subscription Manager or its dependencies are not ready.
     - Stop the xApp; verify RIC readiness, Redis/DBaaS, SubMgr, and E2;
       then rerun.
   * - No E2 connection / connection refused
     - The gNB is not associated with E2Term.
     - Restart RIC, wait for readiness, then restart gNB and confirm E2AP
       completion.
   * - No ``tun_srsue``
     - UE is not attached or its namespace/tunnel is stale.
     - Stop UE, clean stale runtime state if needed, restart UE, and wait
       for the PDU session.
   * - ``Address already in use``
     - A previous xApp still owns fixed REST/RMR ports.
     - Stop the xApp and wait before rerunning. Restart RIC only if the
       stale owner persists.
   * - ``ModuleNotFoundError``
     - A dependency is not detected or declared.
     - Add ``# SoloRAN-Requires``, upload the updated file, and run again.
   * - Python traceback
     - xApp code error.
     - Use the final exception and referenced line number to fix the code,
       upload it, and rerun.
   * - Invalid license during first RIC start
     - External registry/cache seed problem.
     - Do not modify the image. Retain the terminal log and contact the
       SoloRAN administrator.


Full Recovery Sequence
======================

.. warning::

   **Clear is not Stop.** It only removes text from the browser terminal.
   It does not stop an xApp, iPerf, ping, UE, gNB, Core, RIC container, or
   OpenStack instance.

When logs are stale, subscriptions return ``503``, or values do not recover,
use this controlled reset:

1. Click **Stop xApp** and wait for the stopped state.
2. Click **Stop iPerf** and **Stop Ping** if those workloads are active.
3. Click **Stop UE** and wait for ``srsUE stopped``.
4. Click **Stop gNB** and wait for ``gNB stopped``.
5. Click **Stop RIC** and wait for ``Near-RT RIC stopped``.
6. Click **Stop Core** and wait for ``Open5GS Core stopped``.
7. Use **Cleanup All** only if a namespace, tunnel, container, or process
   remains. It is not required between healthy experiments.
8. Restart manually: **Core → RIC → gNB → UE**, waiting for every
   :ref:`readiness gate <soloran-readiness>`.
9. Validate ping, stop it, then start the desired iPerf profile.
10. Run the xApp only after E2 and UE readiness evidence is present.


Stop, Delete, and Sign Out
==========================

End an Experiment Without Deleting VMs
--------------------------------------

1. Stop the xApp and wait for its stopped state.
2. Stop iPerf/ping, UE, and gNB.
3. Stop RIC and Core.
4. Sign out through the profile menu. The OpenStack instances remain
   deployed for a later session.

Delete Deployed Infrastructure
------------------------------

SoloRAN deployments are **Heat stacks**. Deleting only the Nova instance
can leave ports or other stack resources. To remove a deployment completely,
delete the stack that owns it:

1. Stop runtime components in the reverse order described above.
2. In Horizon, open **Project → Orchestration → Stacks** in the correct
   project.
3. Match the RAN or RIC stack to the instance name. Inspect its
   **Resources** tab if uncertain.
4. Choose **Delete Stack**. Confirm only after verifying the target;
   stack deletion cannot be undone.
5. After disassociation, release an unused floating IP to the project pool
   when policy permits.

Delete an Uploaded xApp
-----------------------

1. Stop the xApp if it is running.
2. Select an xApp card uploaded by your account.
3. Choose **Delete selected xApp** and confirm the prompt. You can delete
   your own private or public xApp, never another user's public xApp or a
   system xApp.


Troubleshoot Deployment Failures
================================

.. list-table:: Deployment troubleshooting
   :header-rows: 1
   :widths: 27 28 45

   * - Failure
     - Likely area
     - What to collect or do
   * - Image missing or not visible
     - Glance sharing/visibility
     - Confirm the prepared image is visible to the current project; ask
       the image owner or administrator.
   * - ``BUILD`` / ``Spawning`` for an unusually long time
     - Capacity, image transfer, or compute scheduling
     - Wait for the testbed's normal window. Inspect Heat **Events** and
       the instance **Task**. Avoid repeated deployments.
   * - ``Exceeded max scheduling attempts``; ``Binding failed for port``
     - Neutron port binding / host scheduling
     - Capture instance ID, port ID, timestamp, project, and fault. Contact
       the OpenStack administrator.
   * - No valid host / quota exceeded
     - Flavor capacity or quota
     - Choose an approved flavor/availability zone or request a quota and
       capacity review.
   * - SSH timeout after **Active**
     - Floating IP, security group, or route
     - Verify the floating-IP-to-port mapping and TCP port ``22`` policy
       from the SoloRAN management network.
   * - ``401`` / resources disappear
     - Expired or incomplete session
     - Use **Profile → Manage session**, reconnect OpenStack, and refresh
       resources.


Experiment Record and Support Checklist
=======================================

Keep a record of the following values for each experiment:

.. list-table:: Experiment worksheet
   :header-rows: 1
   :widths: 40 60

   * - Record
     - Values to capture
   * - User / OpenStack project
     - Your user identity and project name.
   * - RIC instance / fixed IP / floating IP
     - Instance name and both assigned addresses.
   * - RAN instance / fixed IP / floating IP
     - Instance name and both assigned addresses.
   * - Images and flavors
     - Image and flavor used for each VM.
   * - Readiness timestamps
     - Core, RIC, gNB, and UE readiness times.
   * - iPerf configuration and result
     - Bandwidth, duration, and measured result.
   * - xApp identity
     - Name, visibility, filename, and version.
   * - E2 / KPM configuration
     - E2 node ID and requested KPM metrics.
   * - Failure details
     - First error line and timestamp.

A useful support report includes the time in **UTC**, your project,
instance/stack/port IDs, the control clicked, the last successful readiness
gate, and the first error line. Include Heat **Events** for deployment
faults. Follow the :doc:`Support Guide <support>` to report an issue.

Never include an authentik password, Keystone password, session cookie,
token, SSH private key, or full environment file.


Security and Responsible Use
============================

* Upload only Python code and dependencies you trust. The xApp runner is a
  controlled research runtime, not a sandbox for hostile code.
* Keep xApp visibility private until the code is ready to share. Public
  visibility does not transfer deletion rights to other users.
* Use the smallest security-group source ranges and ports permitted by
  testbed policy.
* Treat 30 Mbit/s as a stress/DoS profile. Run it only with authorization
  and stop immediately if it affects shared infrastructure.
* Keep experiments within your project. Coordinate explicitly before using
  a shared RIC or shared dataset.
* Delete unused Heat stacks and release unused floating IPs to conserve
  shared capacity.

For CCI-specific quotas, availability zones, image access, and network policy,
follow current testbed administrator guidance when it differs from generic
OpenStack documentation.
