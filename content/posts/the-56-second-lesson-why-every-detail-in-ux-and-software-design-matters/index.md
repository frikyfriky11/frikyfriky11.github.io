+++
date = 2025-09-23T21:05:19+02:00
title = "The 56-Second Lesson: Why Every Detail in UX and Software Design Matters"
description = "A personal story about a parking ticket and a 56-second overage serves as a powerful reminder of how small design decisions in software can have significant real-world consequences and erode trust."
authors = ["Stefano Previato"]
tags = ["UX", "software design", "user experience", "product management", "empathy"]
categories = ["Productivity", "Career"]
+++

I had a frustrating, but ultimately enlightening, experience recently that got me thinking deeply about the decisions we make as developers and designers. It's a small story, but it’s a perfect microcosm of how easily we can fail our users if we're not paying attention to every detail.

My wife, my baby boy, and I were in the city center for a few commissions. We pulled into a multi-story car park and took a ticket at **10:38 AM**. We walked around, did what we needed to do, and came back to the car a little over an hour later. My mental math was simple: we were in the car park for about an hour, which should be **€2**, as the parking fee is €2 per hour.

I walked up to the pay machine, which, as is common, only had a simple display and a slot for the ticket. I inserted the ticket, and the screen lit up with the amount to pay: **€4**.

I was immediately confused. On the ticket itself, it said our entry time was 10:38 AM, and the time we were now paying was 11:38 AM. It didn't make any sense. One hour is €2. How was I being charged for two hours?

Frustrated, I paid the €4 and then went to the customer service desk on the ground floor to get an explanation. I asked the attendant manager why I was being charged €4 when on the ticket I could see that the entry time was 10:38 AM and the exit time was 11:38 AM. He checked the system and, after a brief pause, responded with a shrug. "You probably went over by a couple of seconds," he said.

"Is the parking paid by the second?" I asked, incredulous.

He checked again. "Yeah, let me check... you presented the ticket for payment at **11:38:56 AM**."

That's right. Fifty-six seconds over the minute mark. Because I was **56 seconds** past the one-hour mark, the system automatically rounded up and charged me for the entire next hour. My €2 fee became €4 for a grand total of **56 extra seconds** of parking.

### The Broken Record and the Missing Second Hand

This story isn't about the €2. That's a rounding error in the grand scheme of things. This is a story about the complete failure of the user experience.

Think of it like a broken record player. A good record player gives you a clean, clear playback of the music you want to hear. A bad one, however, skips and distorts the sound, often leaving you with a jarring, unpleasant experience. In this scenario, my user experience was a broken, skipping record.

The first skip was the fee structure itself. When a service charges you by the second, but rounds up to the next full hour, it feels fundamentally dishonest. It's like a baker selling you a single cookie but charging you for an entire dozen because you took a bite. The value exchange is completely unbalanced. You're paying for 59 minutes and 4 seconds of a service you're not even using.

But the biggest skip, the one that makes the whole record player sound like it’s been dragged across concrete, was the **pay machine itself**.

When I put my ticket in, the machine only showed me the amount to pay. It didn't display the entry time, the current time, or the exact moment my first hour expired. If it had been honest and transparent, it would have shown me the entry time and the exit time, including the seconds. **If you're going to charge me by the second, I have a right to see the seconds.**

The attendant's response was a second skip. "You know, not everybody goes around looking at the clock for the exact second." This is a perfect example of a company making assumptions about how its users should behave. Maybe I don't look at my watch. But what if I wanted to? What if I'm on a tight budget and want to maximize every single second I'm paying for? The design assumes a certain level of user behavior and, in doing so, alienates those who don't fit that mold.

### The Architect and the Foundation

Software development and design are like architecture. We're not just building walls and putting a roof on top. We're laying the foundation, designing the flow of movement between rooms, and making sure the light from the windows creates a comfortable, inviting space. Every detail matters, from the structural integrity to the color of the paint.

In the car park story, the developers and designers were so focused on building the walls (the core functionality of the parking system) that they forgot about the foundation and the layout. They decided that a simple display was enough. The difference between `exitTime.format('hh:mm')` and `exitTime.format('hh:mm:ss')` might seem trivial on a screen, but in the real world, it's the difference between transparency and a hidden fee.

This experience was a powerful reminder that every line of code, every pixel on a screen, and every decision we make in the product development process has a real-world impact. It's not just about a functioning feature; it's about the **user's experience** of that feature. Did we create a sense of trust? Was the information transparent? Did we consider how different people might use the system?

As developers and designers, we have a responsibility to be thoughtful and empathetic. Our work isn't just about building things; it's about building trust. It's about creating systems that are fair, transparent, and easy to understand. We must always remember that our users are not just data points or statistics. They're people with their own lives, their own frustrations, and their own limited time.

My parking ticket story is just one small example, but it's a perfect illustration of how a lack of empathy and attention to detail can create a terrible user experience. Let's make sure we're always looking for the "56-second lessons" in our own work and building software that feels honest, not like a broken record.
