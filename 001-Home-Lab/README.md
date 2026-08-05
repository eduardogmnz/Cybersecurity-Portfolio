\# Project 001 – Personal Cybersecurity Home Lab



\## Overview



This project documents the design, implementation, and ongoing development of my personal cybersecurity home lab. The lab serves as a dedicated environment for developing hands-on experience with system administration, networking, vulnerability assessment, penetration testing, digital forensics, and incident response while providing a safe platform for testing security tools and techniques.



The environment is hosted on a dedicated GEEKOM IT15 Mini PC running Windows 11 Pro with Oracle VirtualBox as the virtualization platform. Administrative access is performed remotely from a Lenovo laptop using Microsoft Remote Desktop (RDP), allowing the lab to operate independently as a dedicated virtualization host.



The lab currently includes multiple virtual machines that simulate a small enterprise environment, including Kali Linux, OpenVAS, Windows 11, Windows 7, and Metasploitable2. A dedicated Windows 11 Golden Image is maintained as a deployment template to support rapid provisioning of new virtual machines, while recovery baselines, snapshots, and offline backups provide a repeatable and recoverable testing environment.



This home lab serves as the foundation for every project contained within this portfolio. Future projects—including network enumeration, packet analysis, vulnerability management, exploitation, memory forensics, and digital investigations—are performed within this environment and build upon the infrastructure documented here.



As my cybersecurity knowledge and technical skills continue to grow, this lab will evolve to incorporate additional technologies, services, and defensive capabilities while maintaining a structured, well-documented, and repeatable environment for learning and experimentation.



\## Lab Architecture



The home lab is designed as a dedicated virtualization environment that provides a controlled platform for learning, testing, and validating cybersecurity concepts. The architecture separates physical infrastructure, virtual systems, storage, and recovery components to create a repeatable environment for security research and hands-on experimentation.



The environment is centered around \*\*CYBERHOST1\*\*, a dedicated Windows 11 Pro virtualization host running Oracle VirtualBox. Administrative management is performed remotely from a Lenovo laptop using Microsoft Remote Desktop (RDP), allowing the host to operate independently while simplifying day-to-day administration.



Within VirtualBox, multiple virtual machines simulate systems commonly encountered in enterprise environments, including attacker workstations, vulnerable targets, Windows endpoints, and vulnerability management platforms. Network connectivity is configured to provide both isolated internal communication and Internet access where required for testing.



To support repeatable deployments and rapid recovery, the lab incorporates a dedicated Windows 11 Golden Image, VirtualBox snapshots, recovery baselines, locally stored virtual machine files, and offline backups maintained on external storage.



Figure 1 illustrates the overall architecture of the cybersecurity home lab.



<p align="center">

&#x20; <img src="Assets/CyberLab-Architecture-Diagram-v1.png"

&#x20;      alt="Cybersecurity Home Lab Architecture"

&#x20;      width="1000">

</p>









\## Objectives



The primary objectives of this home lab are to:



\- Develop practical cybersecurity and system administration skills through hands-on experience in a controlled environment.

\- Build proficiency with security, networking, vulnerability assessment, penetration testing, and digital forensics tools.

\- Simulate common enterprise systems and security scenarios using dedicated virtual machines.

\- Provide a repeatable platform for network analysis, vulnerability testing, exploitation validation, incident investigation, and forensic analysis.

\- Maintain standardized deployment, recovery, and backup processes that support reliable lab operations.

\- Produce clear, professional technical documentation for the projects contained within this portfolio.

\- Expand the environment over time as new technologies, tools, and security concepts are introduced.











