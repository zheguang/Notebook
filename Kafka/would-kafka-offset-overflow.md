# Would Kafka Offsets overflow?

This is one of my Kafka Back to Basics notes.

I find my mind wander into a problem probably as old as Greek: will we run out of numbers if we keep counting up?

Kafka assigns each message a number. Message one, two, three, and so on. 
Different messages get different numbers, so we the reader can talk about message `N` without being confused with message `M`.

Kafka also writes messages one after another, as in a log.  Okay there are also topics and partitions to separate unrelated messages. But let's not get too deep into this.  Let's just say we have a log and each message gets a number so we can refer to each message by a unique number, an `offset`. 

Now, computers are finite device. Kakfa's number is of 64-bit long, and no more. And we'd like to keep Kafka running forever. If we keep a log as long as, say human history, when will we run out of numbers? Will it last until the sun goes out?

Let's get down to the numbers. Data type: 64-bit Long. It's signed, so only 63 bits are used to enumerate.  Like ancient Greek, we don't count in the negative, so we are left with $9,223,372,036,854,775,807$ ($2^{63} - 1$) as the last message we can give number to.  That gives us 9.2 quintillion (on the order of $10^{18}$), one followed by eighteen zeros) messages.

How long does it take to get to there?
  - At 1 million messages/second: $\approx 292$ thousand years
  - At 1 billion messages/second: $\approx 292$ years
  - At 1 trillion messages/second: $\approx 292$ days

292 thousand years... Curiously, that's about when homo sapiens, or modern humans, appeared in Africa.  If our ancestors started using Kafka with, say a million folks logging a signle message per second, we are about the run out now (well, we're more than one million strong now).

Where do we store quintillion messages anyways?  Even if each message were just one byte, it would be around eight exabytes.  Currently, total disk storage capacity on Earth is exceeding hundreds of exabytes.  So that's not too bad.  Of course, Kakfa also chops the older portion of the log, by way of retention policy.

To help human race, luckily, Kafka has built-in overflow detection, and the solution is to split the log into more segments [1], and start counting these segments as a base number first, before counting the messages inside them.

This is analogous to how humans count more than ten fingers.  When we count to more than ten, we run out of fingers, but we reuse the ten fingures, just remembering we're now counting how many passed ten.  In other words, we use base `10`, then we can reuse the number `3` as in `3` and `13` and `23`. 

In summary, what happens if overflow occurs is:
  - Kafka detects offset overflow conditions
  - Automatically splits problematic segments
  - Throws specific overflow exceptions
  - System continues operating with split segments


# References
1. [KAFKA-6264](https://issues.apache.org/jira/browse/KAFKA-6264)

