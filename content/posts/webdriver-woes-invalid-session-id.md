---
title: "webdriver woes: invalid session id"
date: 2022-12-01T11:41:13-04:00
draft: false
---

If you're using selenium webdriver at any sort of scale, you'll eventually see some pretty weird stuff.

I'm currently doing a large web crawl project using selenium, and about a day into the crawl, all of my instances suddenly crashed with `selenium.common.exceptions.InvalidSessionIdException: Message: invalid session id`.

This error can happen for legit reasons, like in [this](https://stackoverflow.com/questions/70764346/python-selenium-error-invalid-session-id) SO question/answer; if you close the last open window, then try to access something in the window, you'll get an `invalid session id`, which is exactly what you would expect in this situation.

But that wasn't what was happening for me; I wasn't calling `driver.close()` until the end of the crawl, so the page should definitely be open and accessible.

[This](https://stackoverflow.com/questions/57463616/disable-dev-shm-usage-does-not-resolve-the-chrome-crash-issue-in-docker) SO question/answer gets closer.  By default, chrome uses `/dev/shm` and it's possible that you can exceed the available memory, which also apparently results in an `invalid session id` error.  I monitored `/dev/shm` size during the crawl just by manually checking `df -h` and it never got close to exceeding the limit.  I also figured I should monitor my RAM and disk usage in general, and again, I was never anywhere close to the limits of the machine.

Here's the anti-climatic resolution to the story: I never discovered what was causing the exception.  I ended up catching the `InvalidSessionIdException`, putting the URL being scraped back into the queue, killing the underlying chrome instance using `pkill -f chrome` (remember, calling `driver.close()` or `driver.quit()` in this instance would just cause another `InvalidSessionIdException`), then continuing on with the crawl.

I was banging my head against the wall trying to figure out the source, but I ended up just shrugging my shoulders and just trying again if the error occurred...and it worked.  I did see `InvalidSessionIdExceptions` in my logs, but now I was catching them and trying to scrape the same URL again, and it would work the next time.

selenium is great, but this one was a mystery I won't be solving!