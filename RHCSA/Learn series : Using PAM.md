
### Table of contents

- [Introduction](#introduction)
- [Concept](#concept)
- [How it works](#how-it-works)
- [Configuring demo service using systemd timer](#configuring-demo-service-using-systemd-timer)
- [Common mistakes and debugging tips](#common-mistakes-and-debugging-tips)
- [Cleanup and key takeaways](#cleanup-and-key-takeaways)

### Introduction

`systemd.timer` is the modern Linux replacement for many cron use cases.
It integrates scheduling directly with systemd, giving you better logging, dependency handling, and control.

This tutorial walks through a simple but complete example:

A `hello.service` that writes a timestamped message.

A `hello.timer` that runs it periodically.
