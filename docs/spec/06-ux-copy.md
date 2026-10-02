# 06 UX copy: English catalog, How it works and FAQ

| | |
|---|---|
| Part of | Spec 06 (`06-ux.md`), sections 12.2 and 12.3. Normative. |
| Date, status | 2026-09-24, revision 2 (after the cross-spec review). Ready for build. |
| Rules | Spec 06 section 12.1: tone and lengths, UX-COPY-1 banned words, UX-COPY-2 bonus wording, UX-COPY-3 ad disclosure, UX-COPY-4 no ad promises and the `offer` select. Keys are `area.element.state` in `packages/ui/src/i18n/en.json`; ICU MessageFormat; `{duration}`, `{date}`, `{time}`, `{score}` and `{best}` arrive preformatted (UX-I18N-2). CI fails on em dash characters, banned words, unbalanced ICU and keys referenced by spec 06 but missing here |

## 12.2 English catalog (ICU; `{duration}`, `{date}`, `{time}`, `{score}`, `{best}` are preformatted)

```ini
[nav]
games = Games
leaderboards = Leaderboards
me = My Playground
how = How it works
rules = Official Rules
back = Back

[lobby]
title = Playground
subtitle = {appMode, select, web {Play quick games. Climb the weekly leaderboards. Win P2E Points.} other {Play quick games. Climb the weekly leaderboards.}}
continue = Continue
climb = Where you can climb
all = All games
empty = New games are on the way. Check back soon.
error = Couldn't load the Playground. Try again.

[tries]
summary = {state, select, fresh {{perGame, plural, one {# free try} other {# free tries}} in every game today} other {{left, plural, =0 {No free tries left today} one {# free try left in {games, plural, one {# game} other {# games}} today} other {# free tries left in {games, plural, one {# game} other {# games}} today}}}}
resetsIn = Free tries reset in {duration}
plusHint = {appMode, select, web {Plus members get {plusFree} free tries per game every day. You can also get Plus with P2E Points.} other {Plus members get {plusFree} free tries per game every day.}}
plusLink = Learn about Plus
guest = {free} free tries per game, every day. Log in to play.
freeOf = {left, number} of {allowance, plural, one {# free try} other {# free tries}} left today
runsLeft = {left, plural, =0 {No more ranked runs of {game} today} one {You can play # more ranked run of {game} today} other {You can play # more ranked runs of {game} today}}

[week]
title = This week
endsIn = Ends in {duration}
rewardsUpTo = Up to {amount, number} P2E Points in rewards
closingBanner = The week ends in {duration}. Runs started before the end still count.
newWeek = Week {week} has started. New leaderboards are open.
checking = Week {week} is over. We're checking the final scores. Rewards arrive by {time}.
paying = Week {week} results are final. Rewards are being added.
delayed = Week {week} rewards are running late. They're safe and on the way.

[card]
best = Best {score}
notPlayed = Not played yet
new = New
paused = Paused for updates
comingSoon = Opens {date}
freeLeft = {count, plural, one {# free try} other {# free tries}}
adReady = Ad try ready
freeNone = Free tries back in {duration}
ceiling = Done for today
top100Needs = Top 100: {score}
rank = #{rank}
reward = +{points, number} if the week ended now

[teaser]
seeAll = See all
you = You: #{rank}, {trophies, plural, one {# Trophy} other {# Trophies}}

[start]
free = Play (free try)
freeHelper = {left, plural, one {# free try left today.} other {# free tries left today.}} A try is used when your run starts.
adReady = Play (ad try)
adReadyHelper = Your ad try for this game is ready. Use it by {time}.
points = Play for {cost, number} points
pointsHelper = Balance {balance, number}. Points are used when your run starts.
ad = Watch an ad for 1 try
appPromo = Get more tries in the PlayToEarn app
moreTries = Get more tries
login = Log in to play
practice = Practice (not ranked)
practiceHelper = Practice runs are unlimited and never count for leaderboards.
offline = Play (offline)
reload = Reload
endOther = End the other run

[gate]
ceiling = You've played {ceiling} ranked runs of {game} today. Come back after the reset in {duration}.
outOfTries = You're out of tries for {game} today. Free tries reset in {duration}.
otherGames = {count, plural, one {You still have free tries in # other game.} other {You still have free tries in # other games.}}

[stage]
loading = Loading {game}...
tap = Tap to start
click = Click or press Space to start
starting = Starting your run...
getReady = Get ready: {hint}
cost.free = Uses 1 free try
cost.points = Uses {cost, number} points
cost.ad = Uses your ad try
cost.practice = Practice, not ranked
pause = Pause
paused = Paused
resume = Resume
endRun = End run
scoreSoFar = Score so far: {score}
pausesLeft = {count, plural, =0 {No pauses left in this run} one {# pause left in this run} other {# pauses left in this run}}
challengeTitle = Quick check before your run
challengeBody = This helps keep the leaderboards fair.
sound = Sound
music = Music
fullscreen = Full screen
exitFullscreen = Exit full screen
practiceChip = Practice

[game]
endRunHint = Ending the run saves your score.
endRunConfirm = End this run? Your score so far ({score}) will be saved.
leaveConfirm = Leave the game? Your run will end and your score so far will be saved.
leave = Leave
stay = Keep playing
rotate = Turn your phone upright to keep playing.
offlineNote = You're offline. Your score will be sent when you reconnect.
loadFailed = The game didn't load. No try was used.
crashed = The game stopped. Your score so far ({score}) was saved.
crashedNoScore = The game stopped before your run started. No try was used.

[pay]
title = Out of free tries for {game}
body = Free tries reset in {duration}.
notEnough = You need {cost, number} points. You have {balance, number}.
earn = How to earn points
limitReached = You've reached your daily limit of {limit, number} points. You can change it in Settings.
dailyCap = You've used today's {count, number} points tries across all games. More after the reset in {duration}.
confirmTitle = Use {cost, number} points for 1 try of {game}?
confirmBody = Your balance goes from {from, number} to {to, number} when your run starts. Only your best score of the week counts.
dontAskToday = Don't ask again today
confirm = Use {cost, number} points
notNow = Not now
failed = Couldn't complete the payment. No points were used.
balanceChanged = Your balance changed. You now have {balance, number} points.
priceChanged = The price of a try changed to {cost, number} points. Please confirm again.

[app]
body = In the PlayToEarn app, short ads (when available) can give you up to {count, plural, one {# more try} other {# more tries}} in {game} today.
android = Get the Android app
ios = Get the iPhone app
qrAlt = QR code that opens {url}

[ad]
disclosure = Watch a short ad to get 1 try for {game}. Closing the ad early means no try. {left, plural, one {# ad try left today.} other {# ad tries left today.}}
disclosureNoBoard = Runs from ad tries count for your personal best only, not for leaderboards or rewards.
loading = Loading ad...
confirming = Confirming your try...
granted = 1 try added for {game}.
pending = Your try is on its way. It usually arrives within a minute.
pendingArrived = Your ad try for {game} is ready.
inProgress = Another ad is still open. Close it first.
checkAgain = Check again
closedEarly = The ad was closed early, so no try was added.
noFill = No ad available right now. Try again in a minute.
noFillPoints = No ad available right now. Try again in a minute or use points.
noAds = No ads are available for tries right now.
timeout = The ad took too long to load. No try was used.
error = Something went wrong with the ad. No try was used.
capReached = You've used all ad tries for today. More after the reset in {duration}.
cooldown = Next ad available in {seconds, number} s.
notSupported = Update the PlayToEarn app to watch ads for tries.

[result]
newBest = New best!
firstScore = Your first score this week is on the board!
top100 = You're in the top 100!
first = You're #1!
holdFirst = Hold it until {weekEnd}.
rank = Rank #{rank}
rankOf = of {total, number} players
topPct = Top {pct}% this week
rankUp = {count, plural, one {Up # place} other {Up # places}}
rankDown = {count, plural, one {Down # place} other {Down # places}}
rankSame = No change
bestCounts = Your best this week: {score} (#{rank}). Only your best counts.
closeCall = So close! Your best is {best}.
ifNow = If the week ended now: {points, number} points (ranks {from} to {to})
nextTier = Score {score} to reach #{rank} ({points, number} points)
toTop100 = Score {score} to reach the top 100
allGames = All-Games #{rank} ({delta, plural, =0 {no Trophy change} one {+# Trophy} other {+# Trophies}})
rewardRank = Reward rank #{rank}. Only qualified, eligible players count for rewards.
qualify = Score at least {score} to qualify for rewards in {game}.
namePrompt = Set a display name on your profile so others know who's climbing.
status.saving = Saving score...
status.verifying = Checking your run...
status.saved = Score saved
status.inReview = In review
reviewBody = Your score is being checked. It appears on the leaderboard if it's confirmed.
status.rejected = We couldn't verify this run, so it doesn't count. Run ID {runId}.
status.retrying = Couldn't save yet. Retrying...
status.savedOffline = Saved on this device. We'll send it when you're back online.
status.tooLate = This run was sent too late to count. The try was used.
loginToSave = Log in again to save your score
savedLater = Your score was saved.
adTryNotRanked = Ad-try runs count for your personal best only.
playAgainFree = Play again (free try)
playAgainAdReady = Play again (ad try)
playAgainPoints = Play again for {cost, number} points
playAgainAd = Watch an ad to play again
leaderboard = Leaderboard
share = Share
shareText = I scored {score} in {game} on the PlayToEarn Playground. Can you beat it?
practiceTitle = Practice run
practice = Not saved to leaderboards.
practiceCompare = Your practice best {score} would be about #{rank} this week.
playRanked = Play ranked ({cost})
guestCta = Log in to save scores like this and win P2E Points.
endedPauseLimit = The run ended because it was paused too long. Your score was saved.
endedTimeLimit = The run reached its time limit. Your score was saved.

[lb]
top = Top 100
around = Around me
caption = {board}, Week {week}, {state}
live = Live
provisional = Provisional
final = Final
updated = Updated {time}
provisionalNote = Final after the review that follows the weekly close.
col.rank = Rank
col.player = Player
col.score = Score
col.reward = Reward
col.trophies = Trophies
col.games = Games
you = You
plus = Plus member
inReview = In review
newScores = New scores. Tap to refresh.
rewardZoneEnd = Top 100 reward zone ends here
top100Needs = The top 100 needs {score}
fewPlayers = Top 100 ({count, plural, one {# player so far} other {# players so far}})
fieldScale = Rewards grow with the number of qualified players. Right now they are {pct}% of the full table.
empty = No scores yet this week. Be the first!
rewardWithBonus = {fixed, number} + {bonus, number} Community Bonus

[overall]
title = All-Games Leaderboard
explainer = Every top 100 finish earns Trophies: #1 gets 100, #100 gets 1. {best, select, all {Trophies from all games add up.} other {Your best {best} games count.}}
scaled = On boards with fewer than 100 qualified players, Trophies are scaled down.
gamesRanked = {count, plural, one {# game ranked} other {# games ranked}}
trophies = {count, plural, one {# Trophy} other {# Trophies}}
unranked = Finish in any game's top 100 to join the All-Games Leaderboard.
tieNote = Ties go to more first places, then more second places, and so on.
showGames = Show games
hideGames = Hide games
notCounted = Not counted

[bonus]
title = Community Bonus
announced = Community Bonus this week: {amount, number} P2E Points
estimate = Community Bonus so far: about {amount, number} P2E Points
growing = Community Bonus: growing this week
final = Community Bonus for Week {week}: {amount, number} P2E Points
explainer = Extra P2E Points for the All-Games top {topN}. The bonus is based on the weekly participation activity of the community.
estimateNote = Updated every hour. The final amount is set when the week ends.
splitTitle = How the Community Bonus is shared
yourShare = {mode, select, estimate {Your share if the week ended now: about {amount, number} points} other {Your share if the week ended now: {amount, number} points}}

[reveal]
title = Week {week} results
youEarned = You earned
added = {points, number} points added to your balance
lineGame = {game}: #{rank}
lineAllGames = All-Games: #{rank}
lineBonus = Community Bonus
pending = Being added
held = Held until {date}
heldReview = Held for review
verifyToClaim = Verify your account by {date} to receive {points, number} points.
claimLapsed = Not verified by {date}, so these points were not added.
forfeited = Not paid after review
reversed = Reversed after review
toRedeem = {points, number} points to your first redemption
playNext = Play Week {week}
seeFull = See full results
nothing = No top 100 finishes in Week {week}. Your closest: #{rank} in {game}, {gap, number} away. New week, new leaderboards.

[claim]
verify = Verify to claim
endsIn = {duration} left to verify
how = Verify your wallet with a signed message, or add an email login to your account.
safety = PlayToEarn never asks you to sign a transaction or to verify anywhere except playtoearn.com.
done = Verified. Your rewards are being added.

[elig]
accountTooNew = Rewards for new accounts are held until your account is {days} days old.
review = Your rewards this week are held for a routine check. We'll let you know the result.
region = Rewards aren't available in your region.
staff = Staff accounts can't receive rewards.
deviceLimit = Only one account per device can receive rewards each week.
age = Rewards are for players aged {minAge} or older who have confirmed their age.
rewardsBlocked = Your account can't receive rewards right now. Contact support if you think this is a mistake.
verifyToClaim = You're in the reward zone. Verify your wallet or add an email login to claim your rewards.

[state]
offline = You're offline. Connect to start a run.
maintenance = The Playground is taking a short break. Back soon!
restricted = Your account can't join leaderboards right now. Contact support if you think this is a mistake.
sessionExpired = Your session ended. Log in again to continue.
outdated = A new version is ready. Reload to play. No try was used.
activeElsewhere = You have a run going in another tab or device. If you end it, its try stays used and its score won't count.
gamePaused = This game is paused for updates. Your scores are safe.
gameNotFound = This game isn't available.
region.playground = The Playground isn't available in your region.
region.points = Points tries aren't available here. Free tries reset every day at 00:00 UTC.
region.ads = No ad tries are available in your region right now.
region.prizes = {why, select, personalBest {This run counts for your personal best only.} hold {Your rewards are on hold while we confirm your region.} other {Weekly rewards aren't available in your region. You can still play and climb the leaderboards.}}
appPrizesOff = In the app, runs count for your personal best only for now. Leaderboard runs are on playtoearn.com.
triesChanged = Your tries for {game} changed on another device.
rateLimited = You're starting runs quickly. Try again in {seconds, number} s. No try was used.
interrupted = Your last run was interrupted. We saved your score of {score}.
interruptedLost = Your last run ended when the page closed. That try was used.
unsupported = This browser can't run the Playground. Try the latest Chrome, Safari, Edge or Firefox.
startFailed = Couldn't start the run. No try was used.
ledgerDown = Points are unavailable for a moment. No points were used.

[me]
tab.week = This week
tab.rewards = Rewards
tab.runs = Runs
tab.spending = Spending
tab.updates = Updates
tab.settings = Settings
spent = Points used on tries: {today, number} today, {week, number} this week
source.free = Free try
source.points = Points try
source.ad = Ad try
status.verified = Verified
status.inReview = In review
status.notVerified = Not verified
status.tooLate = Too late
status.removed = Removed

[empty]
runs = Your runs will show up here.
rewards = No rewards yet. Finish in any top 100 to get P2E Points.
spending = You haven't used points on tries.
updates = No updates yet.

[settings]
sound = Game sound
music = Game music
uiSounds = Button sounds
haptics = Vibration
motion = Reduce motion
motionSystem = Use device setting
motionOn = On
motionOff = Off
confirm = Confirm before using points
confirmDaily = Until I turn it off for the day
confirmAlways = Every time
limit = Daily points limit for tries
limitOff = No limit
limitNone = Don't allow points tries
limitValue = {value, number} points a day
limitPending = Your new limit starts {date}.
notify = Notifications
inApp = In the Playground
email = Email
push = Push

[notice]
resultsReady = Week {week} results are in: you earned {points, number} points!
verifyToClaim = Verify your account by {date} to receive your Week {week} rewards.
droppedTop100 = {game}: the top 100 now needs {score}. Your best is {best}.
weekClosing = {game}: you're #{rank} and the week ends in {duration}.
triesBack = Your free tries are back.
newGame = New game: {game}. The leaderboard is wide open!
scoreRemoved = A score in {game} was removed after review. Tap to see why and how to appeal.
rewardReversed = A Week {week} reward was reversed after review. Tap to see why and how to appeal.
payoutHeld = Your Week {week} reward is held until {date}.
payoutDelayed = Week {week} rewards are running late. They're safe and on the way.

[onb]
1.title = Play quick games
1.body = Every game is endless and gets harder until you miss. You get {free} free tries per game every day ({plusFree} with Plus).
2.title = Beat the weekly top 100
2.body = Only your best score of the week counts. The top 100 of every game win P2E Points.
3.title = Collect Trophies
3.body = Top 100 finishes earn Trophies on the All-Games Leaderboard. The best all-rounders win extra P2E Points and share the Community Bonus.
cta = Let's play
rules = Read the Official Rules
rulesLine = By starting a ranked run you agree to the Official Rules.
coach.tries = Your free tries for today. They reset every day at {time}.

[rules]
title = Official Rules
updated = Last updated {date}
version = Version {version}, in force from Week {week}
appleDisclaimer = Apple is not a sponsor of, or involved in, these contests.
acceptTitle = Before your first ranked run
acceptLabel = I'm {minAge} or older and I accept the Official Rules.
accept = Agree and play

[a11y]
frame = {game} game. Press Space to start. Press Escape to pause.
runStart = Run started. Press Escape to pause.
paused = Paused. Score {score}.
gameOver = Game over. Score {score}.
tryAdded = 1 try added.
pointsUsed = {cost, number} points used. Balance {balance, number}.

[toast]
plusWelcome = Welcome to Plus! You now have {plusFree} free tries per game today.

[common]
retry = Try again
close = Close
cancel = Cancel
copy = Copy link
copied = Link copied
otherGames = Other games
points = {amount, number} points
pointsLong = {amount, number} P2E Points
```


## 12.3 How it works and FAQ

```ini
[how]
1 = Pick a game. Every game is endless and gets harder until you miss.
2 = You get {free} free tries per game every day ({plusFree} with Plus). They reset at 00:00 UTC ({localTime} your time).
3 = {offer, select, webApp {Out of free tries? Play for {cost} points, or get more tries in the PlayToEarn app, where short ads may be available.} pointsAd {Out of free tries? Play for {cost} points, or watch a short ad for 1 try when one is available.} ad {Out of free tries? Watch a short ad for 1 try when one is available.} web {Out of free tries? Play for {cost} points.} points {Out of free tries? Play for {cost} points.} other {Out of free tries? New free tries arrive every day.}} Everyone can play up to {ceiling} ranked runs per game each day.
4 = Only your best score of the week counts. When the week ends (Monday 00:00 UTC), the top 100 of every game win P2E Points.
5 = Every top 100 finish earns Trophies on the All-Games Leaderboard. Its top 100 win extra P2E Points and share the Community Bonus. Rewards arrive about 2 days after the week ends.
course = {policy, select, weeklyCourse {Every ranked run this week uses the same course, the same for every player.} dailyCourse {Every ranked run today uses today's course, the same for every player.} other {Every ranked run gets a fresh course.}}
practiceCourse = {policy, select, dailyCourse {After your first ranked run of a game today, you can also practice on today's course.} other {After your first ranked run of a game this week, you can also practice on this week's course.}}

[faq]
free.q = Is the Playground free?
free.a = Yes. You get {free} free tries per game every day.
try.q = When is a try used?
try.a = When your ranked run starts. If a game fails to load or start, no try is used.
more.q = How do I get more tries?
more.a = {offer, select, webApp {Use {cost} points per try. In the PlayToEarn app, short ads may also give you tries when available. Ads are always optional.} pointsAd {Use {cost} points per try, or watch a short ad for 1 try when one is available. Ads are always optional.} ad {Watch a short ad for 1 try when one is available. Ads are always optional.} web {Use {cost} points per try.} points {Use {cost} points per try.} other {New free tries arrive every day at 00:00 UTC.}}
plus.q = What does Plus change?
plus.a = Plus members get {plusFree} free tries per game every day. Everyone has the same limit of {ceiling} ranked runs per game each day.
best.q = Which score counts?
best.a = Your best verified score of the week in each game. If scores are equal, whoever reached the score first ranks higher.
qualify.q = Why don't I see a reward next to my rank?
qualify.a = Rewards count qualified, eligible players only. Some games need a minimum score, and some accounts need to meet the rules first.
claim.q = Do I need to verify my account?
claim.a = Before rewards are added, your account needs an email login or a wallet verified with a signed message. If you win without one, you have {days} days to verify. PlayToEarn never asks you to sign a transaction.
trophies.q = How do Trophies work?
trophies.a = Every top 100 finish earns Trophies: #1 gets 100, #100 gets 1. Trophies from all games add up.
bonus.q = What is the Community Bonus?
bonus.a = Extra P2E Points for the All-Games top 100. The bonus is based on the weekly participation activity of the community. It's shared by rank as shown in the Official Rules.
review.q = Why is my score in review?
review.a = We check every ranked run to keep leaderboards fair. Some scores get a closer look before rewards are added.
ads.q = Do I have to watch ads?
ads.a = Never. Ads are only in the PlayToEarn app, only when you choose them, and never for Plus members. Closing an ad early means no try.
fair.q = What isn't allowed?
fair.a = Bots, scripts, modified games, more than one account, and exploiting bugs. Scores that break the rules are removed, and you can appeal.
practice.q = What is practice?
practice.a = Unlimited runs on random courses. They never count for leaderboards or rewards.
a11y.q = Can I change sound and motion?
a11y.a = Yes, in Settings: sound, music, reduced motion and vibration. Real-time games need sight and quick input, so some players may find them hard.
```

## Open decisions for owner

None beyond `06-ux.md` (launch languages are decided there).

## Cross-spec interfaces

Defines every English key that spec 06 names. Per-game keys come from the spec 04 game files (section 2); `state.region.*` wording follows spec 07 section 2.1; `bonus.explainer` carries the owner's sentence verbatim (R6.1); `rules.acceptLabel` matches spec 07 CMP-006.

## Concerns for orchestrator

None.
