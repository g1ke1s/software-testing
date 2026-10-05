```
expl session SMS flow - late code / expiry
test env 2, iphone ios 18.2
acc TC-014 bal 1 000 000 daily 0 (checked limits screen ok)

- warmup: 150k -> code -> correct -> ok, normal. reset acc
- try 1: 150 000, continue, code came ~5s, copied it
  home btn, waited (stopwatch) ~130s
  back in app -> code screen STILL THERE no timer msg??
  entered code -> confirm -> SUCCESS screen
  bal 850 000 !!! should be expired
- reset acc. again same thing, 130s bg -> success. 2/2
- did it 3 more times -> 5/5 every time accepted
- if i stay IN app and wait 125s -> expired msg shows ok (so only when backgrounded??)
- tried 200s in bg -> didnt try, ran out of time
- expected: code rejected expired, no money
- severity high i think, maybe prio high too - money goes w/o valid code
  ask PO
- todo: screenrec, build nr, req id

misc:
- SMS text RU has typo "кодд"? check again later
- keyboard covers continue btn on small screen sometimes
```
