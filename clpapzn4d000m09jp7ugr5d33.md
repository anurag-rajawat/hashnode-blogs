---
title: "The LFX Mentorship"
datePublished: Thu Nov 23 2023 04:55:19 GMT+0000 (Coordinated Universal Time)
cuid: clpapzn4d000m09jp7ugr5d33
slug: the-lfx-mentorship
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1700621424253/b3afccc6-9b17-4bd4-a8eb-20efc2cd5f82.png
tags: mentorship, lfx-mentorship

---

# Introduction

**Hello everyone! 👋🏻**

In this blog, I'm excited to share my Linux Foundation Mentorship experience with the KubeArmor project. [KubeArmor](https://github.com/kubearmor/KubeArmor) is a cloud-native runtime security enforcement system. If you're unfamiliar with runtime security, I encourage you to join the [KubeArmor community](https://kubearmor.io) and discover the benefits firsthand. I'm confident you'll be impressed.

# Linux Foundation Mentorship

[LFX mentorship](https://docs.linuxfoundation.org/lfx/mentorship) (in short) is a great opportunity to get involved with the open-source community and learn new skills effectively. The program provides mentees with the opportunity to work with experienced mentors on well-known real-world projects, which is a great way to learn by doing. Additionally, the program offers a variety of resources and support, which can help mentees succeed in their goals.

To apply you need your updated Resume and Cover letter in which you clearly state how you plan to implement the task with a proper timeline and that's all you need.

# Background

I'm a self-driven engineer from Madhya Pradesh, India. My self-learning journey began in July 2021 and to get a feel of how real-world projects work I started contributing to open-source projects in January 2022. Later that year, I was selected for [Google Summer of Code](https://summerofcode.withgoogle.com) with [Eclipse Foundation](https://summerofcode.withgoogle.com/archive/2022/projects/y9hRNsi4) for the [JKube](https://eclipse.dev/jkube/) project. After completing GSoC I got very much interested in the CNCF ecosystem and projects so I decided to take part in LFX mentorship.

As we know Go is the most used programming language for CNCF projects so I learnt the Go programming language and Kubernetes at an intermediate level, but I don't know anything about security, so I decided to get some understanding of it by contributing to some projects related to security and that's where I found two projects one is [KubeArmor](https://github.com/kubearmor/KubeArmor) and another one is [Kubescape](https://github.com/kubescape/kubescape) though they serve different purposes. On checking their proposals I found that Kubescape is not my cup of tea so I applied only to KubeArmor and fortunately got the opportunity to work with the KubeArmor project.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700573994475/2a0a77dd-5e23-462e-a460-ff0eb8a33e83.png align="center")

# Mentorship Journey (Learnings & Challenges)

> **TL;DR**
> 
> **Amazing journey with lots of learning.**

Now comes the part I'm waiting for. Having selected KubeArmor and working for over 12 weeks I would say I loved the journey because every day brought some challenges and opportunities for learning. I still remember when I was setting it up locally and was not able to set it up and was doing everything possible to set it up, this experience proved invaluable as I embarked on my contributions to KubeArmor.

During my mentorship, I focused on extending KubeArmor's support to include [K0s](https://k0sproject.io) and [Red Hat MicroShift](https://microshift.io) platforms. This endeavour successfully enabled KubeArmor to secure workloads running on these edge platforms.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700622071641/e88bacc5-6038-4a83-864f-0267c3dc60ec.png align="center")

My mentors, [Barun Acharya](https://barun.cc), [Rudraksh Pareek](https://delusionaloptimist.github.io), [Ankur Kothiwal,](https://github.com/Ankurk99) and [Anurag Kumar](https://github.com/kranurag7), provided invaluable guidance throughout my mentorship. Our weekly sync-ups ensured that we were aligned on progress and goals. Regular interactions with the community further clarified my priorities, leading to the completion of our first major goal, K0s support, within the first four weeks.

Now moving on to the second giant (Why? Just in a moment) which is MicroShift, first I thought it was also easy just like K0s but it was not, it was more than that and it has a lot of moving parts. One of them is SELinux and due to a bug in the KubeArmor controller code, SELinux prevented it from detecting the LSM as a result KubeArmor enforcement was not working and after extensive debugging, that bug was fixed, and KubeArmor was finally able to secure workloads on MicroShift, and that's why I called it giant.

While adding support for it I learned a lot of things such as:

* Different LSMs such as [BPFLSM](https://docs.kernel.org/bpf/prog_lsm.html), [AppArmor](https://apparmor.net), and [SELinux](http://selinuxproject.org/page/Main_Page) and how they work at the fundamental level.
    
* Gained experience with different platforms like [OpenShift](https://docs.openshift.com), [K3s](https://k3s.io), [MicroK8s](https://microk8s.io) and different container runtimes such as [cri-o](https://cri-o.io), and [containerd](https://containerd.io).
    
* Explored low-level tools used by container runtimes such as [runc](https://github.com/opencontainers/runc), [crun](https://github.com/containers/crun) and CRI storage.
    
* Developed an understanding of [Kubernetes Controller](https://kubernetes.io/docs/concepts/architecture/controller/), [Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), and [OpenShift SCC](https://docs.openshift.com/container-platform/4.14/welcome/index.html).
    

During this period I exponentially expanded my knowledge of LSMs and container technologies and my mentors served as great resources, they helped me in every step. I cannot imagine completing MicroShift support without their expertise and support. I thoroughly enjoyed collaborating and learning from them.

I am thrilled to share that [AccuKnox](https://www.accuknox.com), the company behind KubeArmor, has offered a full-time Software Engineer role. This exemplifies that hard work and dedication do pay off.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700579691913/6a091f08-a2d6-433f-ad21-75bd96f8801c.png align="center")

# Conclusion

I would encourage those who prefer hands-on learning through real-world projects to apply for open-source projects like LFX mentorship and GSoC. I understand that it may seem intimidating at first, but all you need is a willingness to learn and some time. So, don't hesitate to step out of your comfort zone and apply!

Open source has played a pivotal role in my professional development. Participating in GSoC and LFX Mentorship has been a transformative experience, propelling my growth in both technical skills and soft skills.

If you have any feedback, please share it in the comments below. Thank you for reading!

Let's [connect](https://linkfree.io/anurag-rajawat)!