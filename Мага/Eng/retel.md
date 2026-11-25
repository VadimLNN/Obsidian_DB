---
tags:
  - education/graduate
  - languages/english
  - security/robotics
---

# 28 Zero-Day Findings in Under Two Days. Is Robotics Security Keeping Pace?

## VicOne’s AI-assisted research reveals what faster zero-day discovery means for robot safety

VicOne identified an initial set of [28 zero-day findings](https://www.zerodayinitiative.com/advisories/upcoming/) in less than two days using AI-assisted research across software and tools supporting the Physical AI ecosystem. These include projects developed or maintained by [Google](https://spectrum.ieee.org/tag/google), [NVIDIA](https://spectrum.ieee.org/tag/nvidia), and [Meta](https://spectrum.ieee.org/tag/meta), among others.

Based on initial validation, all 28 findings are classified as zero-day vulnerabilities. The count may change following vendor and vulnerability-authority review, and some findings do not yet have CVE tracking numbers.

Many of these tools help teams train, simulate, test, and operate robots. Weaknesses in them could influence the models teams build, the test results they trust, and the instructions sent to robots.

These findings create two demands for robotics teams: responding as [AI helps uncover weaknesses faster](https://vicone.com/blog/how-mythos-is-reshaping-vulnerability-management-what-cisos-need-to-know-about-vulnops/?utm_source=ieee-spectrum&utm_medium=sponsored-content&utm_campaign=2026-11), and [assessing how each one could affect robot safety and operations](https://vicone.com/blog/can-a-robot-do-everything-right-and-still-be-unsafe/?utm_source=ieee-spectrum&utm_medium=sponsored-content&utm_campaign=2026-11).

## What AI changes about vulnerability discovery

Vulnerability discovery involves examining code, tracing how systems communicate, and testing whether a suspected weakness can actually be exploited. AI can accelerate parts of this work, helping researchers analyze more software in less time. Attackers could use similar capabilities.

When the weakness is a **zero-day** — previously unknown and without an available fix — faster discovery can leave teams with little time to respond. While that exposure does not necessarily mean an attack has occurred, teams still need to determine whether they use the affected software and what an attacker could access or change.

## Where the findings sit across the robot lifecycle

VicOne’s findings span three stages of the robotics lifecycle: development and training, simulation and validation, and deployment and runtime. Weaknesses at each stage could affect what a robot learns, how it is tested, and how it is controlled.

The following scenarios show how a compromised component could influence the robot at each stage.

- **Development and training: Hidden changes can reach the finished robot  
    A team downloads a pretrained model to adapt for its robot. Loading a malicious file could give an attacker access to the training system, allowing changes to training data or the model itself. A manipulated model could behave normally during routine testing but respond incorrectly to a particular object, scene, or command after deployment. The risk can enter through a tool the team trusts, before anyone attacks the robot directly.**
- **Simulation and validation: A passing test could give false confidence  
    Importing a compromised scene or robot model could give an attacker access to the simulation environment. With sufficient control, the attacker could alter simulated sensor readings, physical conditions, or test results. A robot could then appear to pass tests that should have exposed unsafe behavior, leading the team to approve it for real-world use on the basis of misleading evidence.
- **Deployment and runtime: Operators could lose visibility or control 
    An attacker could exploit weaknesses to change files that influence [motion planning](https://spectrum.ieee.org/tag/motion-planning) or send malicious map data that crashes the operator’s monitoring display. Compromising a remote-operation computer could also put the instructions sent to the robot at risk. Depending on the system and its safeguards, these weaknesses could alter robot movement or leave operators unable to see what is happening and respond promptly.

These are potential consequences and not demonstrated outcomes for every finding. Actual effects depend on the attacker’s access, the system’s design, and whether other safeguards prevent the changes from reaching physical behavior.

In a separate demonstration, [VicOne researchers](https://www.linkedin.com/showcase/vicone-lab-r7/posts/?utm_source=ieee-spectrum&utm_medium=sponsored-content&utm_campaign=2026-11) provide a concrete example of a weakness affecting a protective function. The researchers exploited a vulnerability to prevent a robot’s remote-stop command from working. The case shows why teams need to test whether safeguards remain effective when the software supporting them is compromised.


## **unknown words**

|№|Word|Transcription|Pronunciation (Russian)|Translation|
|---|---|---|---|---|
|1|Initial|/ɪˈnɪʃəl/|инИ́шэл|первоначальный, начальный|
|2|Vendor|/ˈvendər/|вЭ́ндэр|поставщик, разработчик|
|3|Demands|/dɪˈmændz/|димЭ́ндз|требования, задачи|
|4|Assessing|/əˈsesɪŋ/|эсЭ́синг|оценивая, анализируя|
|5|Whether|/ˈweðər/|уЭ́зэр|ли, whether|
|6|Accelerate|/əkˈseləreɪt/|эксЭ́лэрэйт|ускорять|
|7|Exposure|/ɪkˈspoʊʒər/|икспО́ужэр|подверженность риску|
|8|Occurred|/əˈkɜːrd/|экЁ́рд|произошло, случилось|
|9|Affected|/əˈfektɪd/|эфЭ́ктид|затронутый, подвергшийся воздействию|
|10|Span|/spæn/|спЭ́н|охватывать, распространяться на|

## retel 
- VicOne identified 28 zero-day vulnerabilities in Physical AI tools in less than two days using AI-assisted research.
- These vulnerabilities can affect robot development, training, simulation, testing, and real-world operation.
- AI can accelerate vulnerability discovery, but attackers can also use similar capabilities.
- The findings span three stages of the robotics lifecycle: development and training, simulation and validation, and deployment and runtime.
- Robotics teams must identify affected software, assess possible risks, and verify that security updates work correctly.
- Cybersecurity helps protect robots and ensure that safety measures remain effective throughout their lifecycle.