---
layout: post
title: "Security Analysis of the Aperture Science Turret Production Line Turret Integrity Arbiter Integrity Judgment Protocol"
categories: blog
summary: "We analyze the behavior of the Integrity Judgment Protocol, which is used by the Turret Integrity Arbiter on the Aperture Science Turret Production Line to determine the integrity of assembled Aperture Science Sentry Turrets. This analysis is primarily an assessment of the security of the Integrity Judgment Protocol."
---

## Abstract

We analyze the behavior of the Integrity Judgment Protocol, which is used by the Turret Integrity Arbiter on the Aperture Science Turret Production Line to determine the integrity of assembled Aperture Science Sentry Turrets. This analysis is primarily an assessment of the security of the Integrity Judgment Protocol.

## Overview

The Turret Integrity Arbiter is a junction on the Turret Production Line where freshly assembled Aperture Science Sentry Turrets are checked against an existing template to determine their integrity with respects to this template. The template is a Sentry Turret which is known to be correctly assembled.

The following file photo shows the Turret Integrity Arbiter in operation.

![](/assets/img/blog/2019/03/turret-control-center.jpg)

A schematic overview of the Turret Integrity Arbiter is provided for your convenience.

![](/assets/img/blog/2019/03/turret-integrity-arbiter.png)

A freshly assembled stock turret arrives at (1). The Turret Integrity Arbiter first queries the template turret at (2) and (3) and then queries the stock turret at (4) and (5) using the same query. If the answer provided by the stock turret at (5) matches the answer provided by the template turret at (3), the stock turret is approved and continues on the Turret Production Line at (7). If the answers did not match, the stock turret is transported to an incinerator at (6).

It should be noted that the template turret is accessible to authorized Aperture personnel through the Turret Control Center, for maintenance and calibration purposes.

## Protocol model

We model the Integrity Judgment Protocol for analysis by ProVerif, a cryptographic protocol verifier in the formal model. The following listing is a simplified version of the communication steps of the Integrity Judgment Protocol.

```
(1)	Arbiter -> Template : query
(2)	Template -> Arbiter : response
(3)	Arbiter -> Stock : query
(4)	Stock -> Arbiter : (id, response)
(5)	Arbiter -> Stock : approved/rejected
```

The following is a ProVerif model the Integrity Judgment Protocol.

We start with the communication channels:

```
free comm_t, comm_f.
private free comm_s.
```

The first public channel represents communication between the Turret Integrity Arbiter and the template turret. The second public channel is used to output any flags which are used in the model. Both of these channels are public and thus available to an attacker of the system. The private channel is used for communication between the Turret Integrity Arbiter and the stock turret which is being judged. This channel is not available to an attacker of the system.

Next is a pair of functions used for the query/response exchange:

```
private fun question/1.
private reduc response(question(x)) = x.
```

The question function derives a query from some seed value. The response reduction function is used to get the original query seed back. Both functions are private and cannot be used by an attacker of the system.

The following are privately known messages representing the approval/rejection responses from the Turret Integrity Arbiter:

```
private free approved.
private free rejected.
```

The following are the flags and attacker queries used in the model:

```
private free flag_stockrejected.
private free flag_defectapproved.

query attacker: flag_stockrejected.
query attacker: flag_defectapproved.
```

The flags are used to check the integrity of the Integrity Judgment Protocol by ProVerif and do not have any relevance to the Integrity Judgment Protocol itself.

The following is the process of the Turret Integrity Arbiter:

```
let arbiter =
    new x;
    let q = question(x) in
        out(comm_t, q);
        in(comm_t, resp_t);
        out(comm_s, q);
        in(comm_s, (id, resp_s));
        if resp_s = resp_t then
            out(id, approved)
        else
            out(id, rejected).
```

The Turret Integrity Arbiter starts by generating a new question. It then queries the template turret (line 1 of the protocol) and receives its response (line 2 of the protocol). It then performs the same query using the stock turret, and also receives the stock turret’s unique identifier. Depending on whether the responses match, the stock turret is given either an approval or a rejection. Communication of the approval/rejection is done over a channel which is the unique identifier of the stock turret. No other process can communicate over this channel, which ensures correct delivery of the approval/rejection message.

With regards to the stock turret, we model both a correctly assembled Sentry Turret and an incorrectly assembled Sentry Turret. The following is the process of the correctly assembled turret:

```
let stock =
    new id;
    in(comm_s, q);
    let resp_s = response(q) in
        out(comm_s, (id, resp_s));
        in(id, command);
        if command = approved then
            0
        else
            out(comm_f, flag_stockrejected).
```

The correctly assembled turret first generates a new, unique identifier. It then receives the query from the Turret Integrity Arbiter and prepares its response. Both the response and the unique identifier of the turret are then sent to the Turret Integrity Arbiter on the secure communication channel. The stock turret then awaits its judgment which arrives in the form of a command on the unique, secure communication channel belonging to the turret’s unique identifier. If the command is a rejection, we output a rejection flag on the special flag channel. If this flag is found by an attacker in some run of the protocol, this means a correctly assembled Sentry Turret has been rejected by the Turret Integrity Arbiter.

The following is the process of the incorrectly assembled turret:

```
let defect =
    new id; new resp_d;
    in(comm_s, q);
    out(comm_s, (id, resp_d));
    in(id, command);
    if command = approved then
        out(comm_f, flag_defectapproved)
    else
        0.
```

This process is similar to that of the correctly assembled turret. The difference is that the incorrectly assembled turret is not able to correctly answer the query by the Turret Integrity Arbiter, and simply replies with some random fresh variable instead. Lastly, if the defect turret receives an approval judgment, we output an approval flag on the special flag channel, meaning an incorrectly assembled Sentry Turret has been approved by the Turret Integrity Arbiter.

The following is the process of the template turret:

```
let template =
    in(comm_t, q);
    let resp_t = response(q) in
        out(comm_t, resp_t).
```

The template receives a query from the Turret Integrity Arbiter, prepares its response, and sends it back on the public communication channel.

Finally, the process of the protocol is as follows:

```
process (!(arbiter)) | (!(template)) | (!(stock)) | (!(defect))
```

## Protocol analysis

When analyzing the Integrity Judgment Protocol in ProVerif, we obtain the following results:

1.  It **is possible** for a correctly assembled Sentry Turret to be rejected by the Turret Integrity Arbiter.
2.  It **is not possible** for an incorrectly assembled Sentry Turret to be approved by the Turret Integrity Arbiter.

Because communication between the Turret Integrity Arbiter and the template turret can be intercepted and/or modified, it is trivially possible for an attacker to manipulate the system into rejecting correctly assembled Sentry Turrets. An attacker can accomplish this by interrupting the communication between the Turret Integrity Arbiter and the template turret and sending back an incorrect, randomly selected response to the query received from the Turret Integrity Arbiter. Because this manipulated response will be different from the response given by the correctly assembled stock turret, the stock turret will be rejected.

It is not possible for incorrectly assembled turrets to be approved by the Turret Integrity Arbiter, because we are currently working under the assumption that an attacker of the system cannot predict the responses given to the query of the Turret Integrity Arbiter by defect turrets, and thus cannot manipulate the responses given by the template turret to match those. However, consider the following modification to the model:

```
[...]

free resp_d.

[...]

let defect =
    new id;
    in(comm_s, q);
    out(comm_s, (id, resp_d));
    in(id, command);
    if command = approved then
        out(comm_f, flag_defectapproved)
    else
        0.
```

Now that the response given by incorrectly assembled turrets is known to an attacker of the system, we find that **it is possible** for an incorrectly assembled Sentry Turret to be approved by the Turret Integrity Arbiter.

## Conclusion

The Aperture Science Turret Production Line Turret Integrity Arbiter Integrity Judgment Protocol has a serious security and integrity weakness which is found in the fact that the template turret is accessible to an outside attacker and thus sensitive to manipulation. We recommend that either the Integrity Judgment Protocol be reviewed and modified, or the physical security of the template turret be improved.
