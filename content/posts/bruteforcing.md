+++
title = 'Brute Forcing'
date = 2024-02-12T19:45:00Z
draft = false
+++

### Ahhh... brute forcing, my old friend...

Brute forcing. What fun, eh? Can't pick a lock? Knock the door down! If you're reading this article then you've been linked it because someone thinks you should learn why brute forcing is not the hotness you've been lead to believe it is.

So what is brute forcing anyway? In short, it's trying to guess every possible combination of characters and hoping it matches. Let's play a game. I'm thinking of a number beteen 0 and 99. You can have as many guesses as you like. It would make sense to guess 0, then 1, then 2, then 3 etc. Eventually you'd get my number. Congratulations, you've brute forced the answer!


### Brute Forcing Passwords

Burte forcing is, to my knowledge, only talked about in the context of 'cracking passwords'. You have the [hash of a password](https://www.practicalnetworking.net/series/cryptography/hashing-algorithm/) and you want to figure out what the password is. You can't "undo" a hash so what do you do? Let's say the password is a four digit number. If you know what hashing algorithm was used to generate it you can now generate a hash for every combination of four digits and see if it's a match. Four digits is 0000 - 9999, so only 10,000 possible combinations. That shouldn't take too long, right?

...right?

### Entropy

Well I've got some bad news for you. With the four digit number we're only dealing with the numbers 0 to 9. Add in a few more characters, like the letters A through Z in both upper and lower case, and the number of possible combinations explodes. The full alphanumeric space is 62 characters and at four characters long for the 'password' we're trying to crack that'd be 14,776,336 possible combinations. Fourteen _million!_ And that's just four characters long. Maybe you have a nice GPU that can go brrrrr and churn through a million guesses per second. At most it'd take 14 seconds, ezpz gg no re! Except... most passwords are longer than four characters... and don't only use A-Za-z0-9... uh oh!

I'll do the maths for you. A 16 character long password using the full 95 character space (A-Za-Z0-9 plus symbols) is going to be huge. How huge? Do you really want to know? Okay then lol.

```py
95*95*95*95*95*95*95*95*95*95*95*95*95*95*95*95 = 4,401,266,700,000,000,000,000,000,000,000
```

Reckong your GPU can handle that?

### Exhausting The List

So you saved up all your pocket money, you stashed away your allowance, you took on an extra paper route, you stated washing some dishes, and you bought a super duper mega computer. Even at a billion guesses per second this 16 character long password is going to take 4,401,266,700,000,000,000,000 seconds to guess. That's only 139,563,251,522,070 years. Hehe.

So you hacked NASA via `inspect element` and now you have access to their biggest and best computer and that time is not only 139 hours. But still no matches! What gives? I tricked you. The password we're trying to crack is not 16 characters long! Here's the kicker - you'd have to wait for the full list of character combinations to be exhausted before you can know that it's longer than 16 characters. But you don't know if it's 17, or 18, or 32 characters long. Oh fiddlesticks. This whole thing is kind of silly now, isn't it?

### When It Works

So why bother with brute forcing at all? It never works! Well, it sometimes works. Random combinations of characters, digits, and special symbols are hard to remember so humans often use passwords that are more legibile. Something like `password123` is easier to remember than `(i dJ"14N%"`. And sadly human beings are quite predictable. This is where wordlists come into place.

Wordlists are lists of commonly used passwords (such as password123). When cyber security professionals use a brute forcing tool it will try the words in the wordlists first before trying random combinations. This adds more calculations for the comptuer to do (they don't want to retry character combinations they've already guessed from the wordlists) but is sometimes worthwhile. This is still a last resort, though. It's not like a wordlist can contain every single possible password ever otherwise we run into the 149 gabillion jazillion years problem again. Any tools that try to generate passwords for the wordlist will still hit this problem eventually.

### When It Doesn't Work

There are many password managers around today that can help with generating secure passwords. I use [Bitwarden](https://bitwarden.com/). They have a free tier, a web browser plugin, it can be hosted locally, or you can use their cloud. Whilst brute forcing is a ridiculous attack it takes very little to mitigate against it. Take these passwords: `Password!1234567` and `^8o*&D#74zcA7Pgi`. They both use the same mix of character spaces but one will be cracked almost immediately as it will show up in wordlists. You don't need to remember `^8o*&D#74zcA7Pgi` because you've got a password manager to do that for you. Hurrah!

### Conclusion

Password managers can get hacked so read up on the one you choose before using them. Or consider hosting locally. [KeePass](https://keepass.info/) will *keep* your *ass* safe and is 100% locally hosted. What else can I say to pad this out... oh aye, how will you even get password hashes lol. If you're reading this you're probably a kid just starting out. I'm not gonna show you that; I ain't [NetworkChuck](https://www.youtube.com/@NetworkChuck). Love you Chucky xxox
