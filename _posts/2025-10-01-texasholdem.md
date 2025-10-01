---
layout: single
title: "Texas Hold 'Em Cheat Sheet"
date: 2025-10-01
subtitle: 'Creating an intuitive cheat sheet to distill the complexity of winning a game of poker'
excerpt: 'Creating an intuitive cheat sheet to distill the complexity of winning a game of poker'
author_profile: false
tags: [Statistical Modelling]
header:
  teaser: /assets/projects/20251001_texasholdem/TexasHoldEmOdds_10Players.png
---

*tl;dr: Below is a table summarizing your odds of winning a hand of Texas hold 'em at any betting stage, with any hand.  Use it to skunk your next poker night with the power of stats!  Read on for the full process of developing it.*

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/TexasHoldEmOdds_10Players.png)
<figcaption>A single table summarizing the odds of winning a hand of Texas hold 'em at each betting stage based on the cards in your hand</figcaption>


## Intro
A quick palette cleanser after the hellscape of Gerrymandering!  After losing all my cents at a boys night of penny poker, I found myself wishing I had a statistical reference sheet to know how "good" my hand actually was at each stage of the game.  It felt like it made statistical sense to fold *every* hand at the pre-flop stage unless you got pocket aces, since you "probably weren't going to get anything good".  However, I realized everyone at the table is in the same boat, so was curious about a way to see if the *relative* odds were in your favour, and if this could be distilled into one simple-to-reference chart.


## Background
As a quick overview/refresher, Texas Hold 'Em is a version of poker where each player is dealt two cards, and five additional cards are dealt face up in the center ("the river").  Players compete to form the best five-card poker hand they can using the seven available cards (two in their hand, plus five in the communal river).  Ranking order of possible hand types is:

1. **Royal Straight Flush** ("10,J,Q,K,A" all of the same suit)
2. **Straight Flush** (any five sequential cards of the same suit)
3. **Four of a Kind** (like it sounds, any four cards with matching face values)
4. **Full House** (three cards of one matching face, plus two of another - e.g. "A,A,A,Q,Q")
5. **Flush** (five cards of the same suit)
6. **Straight** (any five sequential cards)
7. **Three of a Kind** (like it sounds)
8. **Two Pair** (two pairs of matching face value)
9. **Pair** (two cards of matching face value)
10. **High Card** (none of the above, simply your highest card face value)

If two players have the same type of hand, the hand containing the highest face value wins (suits don't matter).  For straights, aces can be either high or low at the player's discretion, but not both at once (e.g. "K,Q,A,2,3" is not a valid straight).  For simplicity, I'm ignoring "Royal Straight Flush" from here on, since it's just a high-ranking instance of a straight flush (I think people just like to have a name for it).

Cards are not dealt all at once, rather in stages with opportunity ("obligation", really) for betting between each.  First, players are dealt their two personal cards ("the pre-flop"), then three of the communal river cards are dealt ("the flop"), then one more river card ("the turn"), then the last river card ("the river").  There's a final round of betting after "the river", and then players reveal their cards and most of them lose.

Alright, let's get to it!



## Getting the Data
None needed!  What a treat!




## The Analysis
In terms of overall approach, I chewed for a while on if there were some clever statistical shortcuts I could take here to calculate the odds of winning.  In the end I just said screw it and went full brute-force Monte Carlo.  Not necessarily because I didn't think such stats shortcuts existed, but just because the analysis was pretty lightweight and it seemed more pragmatic to just get started.  The elements of five cards being shared between all players, and hands only using the best 5 of each player's 7 available cards added a fair bit of complexity to any high-level statistics approach.


### The Basics
So, the approach was basically to just model a bunch of Texas hold 'em hands against random opponents and measure the outcome.  For each hand, I would measure what each player ended up with and who won, but also calculate the *interim* hands players had at each betting stage so we could track how these evolved into either wins or losses.

First up was modelling the deck and a function to evaluate the best poker hand each player had.  Not much output to show here but... I did that.  Check out the dropdown if you want to see the code and how it works.

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/checkhand1.png)
![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/checkhand2.png)
![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/checkhand3.png)
![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/whowins1.png)
<figcaption>...it works</figcaption>

<details>
  <summary>Click here if you don't believe I can model 52 cards</summary>
  An early version of this code used some beautiful pandas structures with easy-to-read named references and intuitive evaluation order/logic.  Unfortunately, this model was going to get some serious reps under its belt so I had to gut out all the expensive pandas overhead to maximize runtime performance into a reasonable range.  Below is the final version of the script that can handle 1M executions in ~90s.<br><br>
  
  The core approach is to first define the deck of playing cards (face values, suits, numerical rankings), and to define "tags" to allow concise referencing of individual cards.  This was converted to a dictionary to speed up references:

  {% highlight python %}
#first, define the deck
deck = []  #format [tag, face, numrank, suit]

for suit in ['spades', 'clubs', 'hearts', 'diamonds']:
    for n in range(1,14):
        #name the face
        if n==1: face='A'
        elif n==11: face='J'
        elif n==12: face='Q'
        elif n==13: face='K'
        else: face=n

        numrank = 14 if n==1 else n  #call ace 14 for later hi card ease

        tag = str(face) + suit[:1]

        deck.append([tag, numrank, face, suit])

deck = pd.DataFrame(deck, columns=['tag', 'numrank', 'face', 'suit'])
deck = deck.set_index('tag')
deck

deckdict = deck.to_dict()  #need more speeeeed
  {% endhighlight %}


  Next was a function to check the highest-scoring poker hand in a given set of cards.  This logic had to be dynamic to card sets of any length, not just 5 (an early iteration evaluated 5-card combinations independently, but that was slow).  The core idea of this is to, in one fell swoop, evaluate the best existing hand of that "type" (see <strong>Preliminary Hands</strong> for a bit more context on the types I defined.  Hand types are evaluated in priority order to save compute evaluating lesser hands.  There's a bit of a unique approach for each hand type, but these all work primarily off of list comprehension to minimize overhead:

  {% highlight python %}
#make a function to check what you have
def checkhand(lst):  #input is a list of card tags (of any length).  output is [hand, hi card numrank]
    #updating to faster logic from the baby checks
    suits = [deckdict['suit'][card] for card in lst]
    nranks = [deckdict['numrank'][card] for card in lst]
    if 14 in nranks: nranks.append(1)  #account for duality of aces in straights
    
    #evaluate hands in priority order to potentially save computes
    #best straight flush
    bsflush = 1
    for suit in suits:
        matching = [nranks[_i] for _i, s in enumerate(suits) if s==suit]
        for rank in set(matching):
            sflen = len([n for n in set(matching) if rank>=n>rank-5])
            if sflen > bsflush: 
                bsflush = sflen
                sfhicard = rank
    bsflush = min(5, bsflush)
    if bsflush==5: return ['straight flush', sfhicard]
    
    #best x of a kind
    _sortranks = sorted([(k,v) for k,v in Counter(nranks).items()], key=lambda x: (x[1], x[0]), reverse=True)  #can reuse this***   #sort x[1]x[0] - use rank to break ties!
    xhicard, bxkind = _sortranks[0]
    if bxkind==4: return ['four of a kind', xhicard]

    #full house?
    fullhouse=False
    if len(_sortranks)>=2 and _sortranks[0][1]>=3 and _sortranks[1][1]>=2:  #short circuit to prevent errors on a pair preflop
        fullhouse=True
        fhhicard = max([_[0] for _ in _sortranks[:2]])
    if fullhouse: return ['full house', fhhicard]
    
    #best flush
    (_fsuit, bflush) = max([(k,v) for k,v in Counter(suits).items()], key=lambda x: x[1])
    fhicard = max([nranks[_i] for _i,s in enumerate(suits) if s==_fsuit])
    bflush = min(5, bflush)  #don't count >5 (we do technically lose some insight here... if you have 7, someone else has 5)
    if bflush==5: return['flush', fhicard]
    
    #best straight
    bstraight = 1   
    for rank in set(nranks):
        slen = len([n for n in set(nranks) if rank>=n>rank-5])
        if slen > bstraight: 
            bstraight = slen
            shicard = rank
    bstraight = min(5, bstraight)
    if bstraight==5: return ['straight', shicard]

    #3 of a kind?
    if bxkind==3: return ['three of a kind', xhicard]
    
    #2 pair?
    twopair = False
    if len(_sortranks)>=2 and _sortranks[0][1]>=2 and _sortranks[1][1]>=2:
        twopair=True
        tphicard = max([_[0] for _ in _sortranks[:2]])
    if twopair: return ['two pair', tphicard]

    #pair?
    if bxkind==2: return ['pair', xhicard]
    
    #hicard
    return ['hi card', xhicard]
  {% endhighlight %}

  Finally, a quick function to compare <em>multiple</em> hands and evaluate a winner:

  {% highlight python %}
#define hand ranking
handranks = {
        'straight flush': 8,  #royal straight flush is just a really high straight flush - hi card will handle this
        'four of a kind': 7,
        'full house': 6,
        'flush': 5,
        'straight': 4,
        'three of a kind': 3,
        'two pair': 2,
        'pair': 1,
        'hi card': 0
    }

def whowins(lst):  #define a function to compare a list of hands (lst of lsts) and returns winner index
    hands = [checkhand(h) for h in lst]
    scores = [handranks[h[0]] + h[1]/1000 for h in hands]  #posn 2 is handscore
    maxscore = max(scores)
    winners = [i for i, s in enumerate(scores) if s==maxscore]  #check for multiple winners
    return random.choice(winners)  #break ties arbitrarily - comes out in the wash
  {% endhighlight %}

</details>




### Monte Carlo
Okay, with that set up - let's let the robots play!  I had them play games of 2-10 players (allowable table sizes for Texas hold 'em) and they got to play a million hands at each table... lucky guys!  Why a million?  Why not.  Again, not much real output to show at this stage, so here's a pretty picture of the 9M simulations colouring the ones we won:

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/simulationsstarmap.png)
<figcaption>A mesmerizing but useless map of 9 million Texas hold 'em simulations, highlighting hands we won and the table size</figcaption>

<details>
  <summary>Shhh - robots at play</summary>
  It's a loop!  The order of execution doesn't directly follow real world play order, as there were some quicker evaluation methods available by doing later stages earlier and re-using evaluated components.  The robots weren't playing for real money so they didn't mind:

  {% highlight python %}
simpledeck = list(deck.index)  #speed

outcomes = []  
#to maximze memory, each hand will be stored in the following format:
#ONLY "our" hand (player 0) and result is stored, as a single list [player card 1, player card 2, river1,river2,...river5, hand, hicard, nplayers, win?(1/0)]

for nplayers in range(2,11):  #start with just the 10p game
    for _c in range(int(1e6)):  #try 1M sims of each
        playcards = random.sample(simpledeck, 5+nplayers*2)  #to simulate sample without replacement, just identify the cards we'll be playing with upfront and hand them out - random, order doesn't matter

        river = []
        for _ in range(5): river.append(playcards.pop())  #counter intuitive, but we're doing the river FIRST.  random so order doesn't actually matter, and allows rapid access of assigning final outcome by player.  Reveal order can be implied
        
        playerhands = []  #for the sake of syntax, "we" are ALWAYS player 0 (identified solely by index position for memory optimization)
        for p in range(nplayers):
            a,b = playcards.pop(), playcards.pop()  #two cards to each player
            playerhands.append([a,b]+river)  #for each player, they have their two cards plus the river (order matters)
    
        yourhandresult = checkhand(playerhands[0])
        winner = whowins(playerhands)
        youwin = 1 if winner==0 else 0
    
        outcomes.append(playerhands[0] + yourhandresult + [nplayers] + [youwin])  #log the result of just our hand (player 0)
    
        if _c%100e3==0: 
            print(f"{datetime.datetime.now(): %T}: {nplayers} players / {_c:,} iterations complete", flush=True)  #flush to ensure output update in fast loop
            np.savez_compressed("outcomes.npz", outcomes=outcomes)
    
    np.savez_compressed("outcomes.npz", outcomes=outcomes)  #final dump

print("complete")
  {% endhighlight %}
</details>




### Overall Results
Before getting into the full analysis at each betting stage, there's some cool checks we can do on the overall results.  Specifically, checking in on the net likelihood of each hand, as well as its win rate depending on the table size.  On the overall pull rates, we will be happy to confirm that the powers that be were not out to lunch when they decided which hands were most valuable (a straight is indeed more common than a flush).  One notable exception is the single high card, which is actually *less* likely to end up with than both a pair and **two pairs**.  The high-level stats check out on that, but food for thought next time you think you're laughing with pocket aces:

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/pullrates.png)
<figcaption>Relative pull rates of each poker hand in seven-card Texas hold 'em</figcaption>


We can also calculate the win rate of each type of hand, which is dependant on the number of players at the table. As more players are introduced at the table, it becomes more likely someone will beat you and the winning hands become more competitive. For example, "two pair" is practically a shoe-in at a two-person table, but a death sentence at 10:

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/handwinrate.png)
<figcaption>Overall win rates for each poker hand, per Texas hold 'em table size</figcaption>

<details>
  <summary>Full code for nerds</summary>
  With that heavy loop out of the way, we're able to convert back to pandas DataFrames.  Much better:

  {% highlight python %}
#back to df
outcomes = np.load("outcomes.npz")['outcomes']
outcomesdf = pd.DataFrame(outcomes, columns=['c1','c2','r1','r2','r3','r4','r5','hand','hicard','nplayers','wewin'])

outcomesdf[['hicard', 'nplayers','wewin']] = outcomesdf[['hicard', 'nplayers','wewin']].astype(int)
outcomesdf.head(10)


#useless but pretty visual of 9M sims
lst = [int(o[9]) if o[10]=='1' else 0 for o in outcomes]
arr = np.array(lst).reshape(3000,3000)

fig, ax = plt.subplots(figsize=[12,6])
sns.heatmap(arr[::-1], ax=ax, cmap='inferno', cbar_kws={'label':'Table Size'})
ax.set_title("")
ax.set_xticks([])
ax.set_yticks([])
fig.savefig("simulationsstarmap.png", bbox_inches='tight')


#most likely hands
_grp = outcomesdf.groupby('hand').size().sort_values() / len(outcomesdf)
_df = pd.DataFrame(_grp, columns=['Pull Rate'])
_df.index = _df.index.str.title()

fig, ax = plt.subplots(figsize=[1.5,5])
sns.heatmap(_df, annot=True, fmt=".2%", annot_kws={'size':9}, cmap='RdYlGn', cbar=False, vmin=0, vmax=1, linewidths=0.5, linecolor='k')
ax.set_ylabel('')
ax.set_title("Pull Rate by Hand")
fig.savefig("pullrates.png", bbox_inches='tight')


#win rate by n players
_grp = outcomesdf.groupby(['hand', 'nplayers'], as_index=False)['wewin'].agg(['sum', 'size'])
_grp['winrate'] = _grp['sum'] / _grp['size']
_grp = _grp.sort_values('size')

_pvt = _grp.pivot(index='hand', columns='nplayers', values='winrate').sort_values(10, ascending=False)
_pvt.index = _pvt.index.str.title()

fig, ax = plt.subplots(figsize=[8,5])
sns.heatmap(_pvt, annot=True, fmt=".0%", annot_kws={'size':9}, cmap='RdYlGn', cbar=False, vmin=0, vmax=1, linewidths=0.5, linecolor='k')
ax.set_ylabel('')
ax.set_xlabel("Table Size")
ax.set_title("Hand Win Rate")
fig.savefig("handwinrate.png", bbox_inches='tight')
  {% endhighlight %}
</details>




### Preliminary Hands
Okay, so while the summaries of final hands above are interesting high-level statistics, they don't help you much until after you've already gambled your kids' college savings.  I was interested to take these back a step, and see if I could calculate similar win rate statistics for *early game* hands to predict a player's likelihood of winning *before* the final hands are known.  To do this, the Monte Carlo simulation above actually logged more than just the final hand and who won - it also logged the cards each player had access to at each of the earlier betting stages.  This allowed us to go "back in time" and evaluate the hand a player would have been looking at early on, with the future knowledge of what they ended up with and if they won.

The first (arguably biggest) challenge in this step was deciding how on earth to summarize all of the possible things an early hand could become.  If I'm looking at "Queen + Ace" off the draw - with five more cards that could become literally any one of the poker hands above (or nothing at all).  To start, I summarized all the poker hands more simply into "straight/flush-like" (Straight, Flush, Straight Flush) and "x of a kind" (Pair, Three of a Kind, Four of a Kind, Two Pair, Full House).  Each hand in both of these sets are derivatives of each other, and there is a convenient linear hierarchy within each set.

My high-level approach was to, at each betting stage, log every possible hand a player could consider themselves working towards.  For example, the hand "A,A,3,4,J" could be on its way towards any of a Pair, Three of a Kind, Four of a Kind, Two Pair, Full House, or a Straight (plus flush/straight flush if the suits were right).  The categories above let me simplify this a lot by simply saying the hand was "2 cards down the 'x of a kind' road" and "3 cards towards a straight".  The linear hierarchies within each group also let me avoid logging irrelevant options (e.g. there's no point calling three jacks "a pair").  Running each of the 9M hands through this produced something like the following:

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/combinedcropped.png)
<figcaption>Possibilities at each betting stage for one hand.  At the pre-flop it looked like a straight flush ("9,_,_,_,K").  At the flop it looked like either a straight flush ("9,_,J,_,K") or a pair ("J,J").  At the turn it looked like a straight flush (same), a pair (same), or a flush (4 hearts).  Ultimately, it was a pair.</figcaption>

<details>
  <summary>Full code and method</summary>
  This logic picks the data up where the last one left off, and uses the "final hands" dataset as its starting point. The loop logic is very similar to the function used to evaluate final hands above, however it is not restricted to "completed" hands, and allows appending multiple valid results.  The names of hands are also changed to align with the simplified structure outlined above:

  {% highlight python %}
bettingstages = {  #define how many cards were visible to the player at each betting stage
    'preflop': 2,
    'flop': 5,
    'turn': 6,
    'river': 7
}

stageorder = ['preflop', 'flop', 'turn', 'river']
handtypeorder = ['x of a kind', 'two pair', 'full house', 'straight', 'flush', 'straight flush']  #for category sorting later

workingfinalconversion = {  #create a conversion between the final hands and interrim names we give them
    'straight flush': ['straight flush', 5],  #royal straight flush is just a really high straight flush - hi card will handle this
    'four of a kind': ['x of a kind', 4],
    'full house': ['full house', 5],
    'flush': ['flush', 5],
    'straight': ['straight', 5],
    'three of a kind': ['x of a kind', 3],
    'two pair': ['two pair', 4],
    'pair': ['x of a kind', 2],
    'hi card': ['x of a kind', 1]
}

#check each hand
stageoptions = []  #this will store all available options in format [o index, stage, handtype, n completed, hicard nrank]

#assess our hand at each stage, explode out to all "things" we could have been working toward at that stage (one hand can be multiple partial things)
for _i, o in enumerate(outcomes):

    #assess all possible "working hands" at the first 3 stages
    for stage in stageorder[:3]:
        hand = o[:bettingstages[stage]]  #first n cards visible at this stage
        suits = [deckdict['suit'][card] for card in hand]
        nranks = [deckdict['numrank'][card] for card in hand]
        if 14 in nranks: nranks.append(1)  #account for duality of aces in straights
        

        #best flush
        (_fsuit, bflush) = max([(k,v) for k,v in Counter(suits).items()], key=lambda x: x[1])
        fhicard = max([nranks[_i] for _i,s in enumerate(suits) if s==_fsuit])
        bflush = min(5, bflush)  #don't count >5 (we do technically lose some insight here... if you have 7, someone else has 5)

        #best straight
        bstraight = 1   
        for rank in set(nranks):
            slen = len([n for n in set(nranks) if rank>=n>rank-5])
            if slen > bstraight: 
                bstraight = slen
                shicard = rank
        bstraight = min(5, bstraight)

        
        #best straight flush
        bsflush = 1
        for suit in suits:
            matching = [nranks[_i] for _i, s in enumerate(suits) if s==suit]
            for rank in set(matching):
                sflen = len([n for n in set(matching) if rank>=n>rank-5])
                if sflen > bsflush: 
                    bsflush = sflen
                    sfhicard = rank
        bsflush = min(5, bsflush)
                

        #best x of a kind
        _sortranks = sorted([(k,v) for k,v in Counter(nranks).items()], key=lambda x: (x[1], x[0]), reverse=True)  #can reuse this***   #sort x[1]x[0] - use rank to break ties!
        
        xhicard, bxkind = _sortranks[0]

        #full house?
        fullhouse=False
        if len(_sortranks)>=2 and _sortranks[0][1]>=3 and _sortranks[1][1]>=2:  #short circuit to prevent errors on a pair preflop
            fullhouse=True
            fhhicard = max([_[0] for _ in _sortranks[:2]])

        #2 pair?
        twopair = False
        if len(_sortranks)>=2 and _sortranks[0][1]>=2 and _sortranks[1][1]>=2:
            twopair=True
            tphicard = max([_[0] for _ in _sortranks[:2]])
        

        #only append meaningfully different possibilities
        if bsflush>1: stageoptions.append([_i, stage, 'straight flush', bsflush, sfhicard])
        if bstraight>1 and bstraight>bsflush: stageoptions.append([_i, stage, 'straight', bstraight, shicard])
        if bflush>1 and bflush>bsflush: stageoptions.append([_i, stage, 'flush', bflush, fhicard])

        if bxkind==4: stageoptions.append([_i, stage, 'x of a kind', 4, xhicard])
        elif bxkind==3:
            if fullhouse: stageoptions.append([_i, stage, 'full house', 5, fhhicard])
            else: stageoptions.append([_i, stage, 'x of a kind', 3, xhicard])
        elif bxkind==2:
            if twopair: stageoptions.append([_i, stage, 'two pair', 4, tphicard])
            else: stageoptions.append([_i, stage, 'x of a kind', 2, xhicard])

        if bxkind==1 and bstraight==1 and bflush==1: stageoptions.append([_i, stage, 'x of a kind', 1, xhicard])  #only hi card


    #straight append the river stage (just to keep data format simple and consistent)
    finalhand = o[7]  #speed, list reference
    finalhicard = o[8]
    stageoptions.append([_i, 'river'] + workingfinalconversion[finalhand] + [finalhicard])
    
    
    if _i%100e3==0: 
        print(f"{datetime.datetime.now(): %T}: {_i:,} outcomes complete", flush=True)
        np.savez_compressed("stageoptions.npz", stageoptions=stageoptions)
        

print("complete")
np.savez_compressed("stageoptions.npz", stageoptions=stageoptions)
  {% endhighlight %}
</details>





The keen observer will notice that this analysis introduced a lot of "duplicates", in the sense that one hand can be many things at one time.  This is fine because we do want to report each of the possibilities, however the current setup would unfairly attribute wins to irrelevant early-game hands (e.g. if a player has a pair of jacks at the flop and goes on to win with a *flush*, it would be remiss to credit that victory as "likely if you have a pair of jacks at the flop").  Again, the two hand groupings are helpful here as they establish (mostly) clear lineages of hands that build towards each other.  As a result, we can ensure that we only attribute *relevant* wins to early game hands:

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/combinedsample.png)
<figcaption>The same possibilities as above - note that only the "x of a kind" hand possibilities were deemed 'relevant' to the final hand</figcaption>

<details>
  <summary>Full code and method</summary>
  Again, with the heavy stuff out of the way we can go back to pandas.  The options logged at each stage are converted to a DataFrame, and then merged back onto the "final hands" dataset to give each row complete "where we are now" vs "where this hand ends up" visibility:

  {% highlight python %}
stageoptions = np.load("stageoptions.npz")['stageoptions']
stageoptionsdf = pd.DataFrame(stageoptions, columns=['outcome_index', 'stage', 'workinghand','n', 'workinghicard'])

stageoptionsdf[['outcome_index', 'n', 'workinghicard']] = stageoptionsdf[['outcome_index', 'n', 'workinghicard']].astype(int)
stageoptionsdf['stage'] = pd.Categorical(stageoptionsdf['stage'], categories=stageorder, ordered=True)  #this will come in handy later
stageoptionsdf['workinghand'] = pd.Categorical(stageoptionsdf['workinghand'], categories=handtypeorder, ordered=True)

stageoptionsdf.head(10)

#confirm every outcome index got at least one log
[o for o in outcomesdf.index if o not in stageoptionsdf['outcome_index']]

#combine back onto final results
optionscombined = stageoptionsdf.merge(outcomesdf, right_index=True, left_on='outcome_index')
optionscombined.head(10)
  {% endhighlight %}


  A mapping was made for each of the unique "working hands", and the "final hands" that we consider relevant (i.e. are derivatives of the working hand).  Quick columns were added to easily add these up later on:

  {% highlight python %}
relevanthands = {  #because we're including ALL possible hands at each point, it's unfair to attribute irrelevant early-game hands to unrelated outcomes
    'straight flush': ['straight flush', 'straight', 'flush'], 
    'flush': ['flush'],
    'straight': ['straight'],
    'x of a kind': ['hi card', 'pair', 'two pair', 'three of a kind', 'full house', 'four of a kind'],
    'full house': ['full house', 'four of a kind'],
    'two pair': ['two pair','three of a kind', 'full house', 'four of a kind']
}

#check if each of the things we were going for actually worked out (i.e. we got the thing we were working on... and won)
optionscombined['relevant'] = [1 if r['hand'] in relevanthands[r['workinghand']] else 0 for _i, r in optionscombined.iterrows()]
optionscombined['workedout'] = [1 if r['wewin']==1 and r['relevant']==1 else 0 for _i, r in optionscombined.iterrows()]

optionscombined.head(10)
  {% endhighlight %}
</details>






### Putting it All Together
With all early-game hands and relevant wins/losses established, the final challenge was how to compile all of this noise into a single easy-to-read chart.  The format I landed on is as follows, with each of the six unique types of poker hands (broken into their two groups) tracked across each betting stage of a hand.  Where relevant, the "percentage completion" of each hand has a unique value.  The probabilities in each box represent the odds that you will go on to win with **that hand** (or a derivative of it).  These values are *additive*, since they only reflect the win likelihood of relevant final hands and lower-derivative hands were not logged at any stage:

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/annotatedsimplified10p.png)
<figcaption>A simplified version of the final summary chart, with win probabilities for each early-stage hand</figcaption>


The last thing to add in here was *ranges*.  Not all hands are equal, and so your odds might look much better if you're working off a pair of **aces** vs. **2s**.  To handle this, the relevant high card was also logged for each of the early-game betting stage hands.  Instead of just grouping all "pairs at the turn" together, we can stratify by them by the relevant high card and actually calculate the exact win odds for a pair of 4s vs. a pair of queens.  In the final chart, these are merely summarized as full ranges.  A player using this chart should interpolate between these depending on what they're holding:

![]({{ site.baseurl }}/assets/projects/20251001_texasholdem/TexasHoldEmOdds_10Players.png)
<figcaption>The final summary chart of Texas hold 'em win probability for every hand, at every betting stage</figcaption>

<details>
  <summary>Full code for nerds</summary>
  Pretty simple groupby -> pivot here.  Values are aggregated by hi card, and then those win rates are aggregated to give us our overall probability range.  A weighted average (irrespective of high card) is maintained throughout to create the color heatmapping in the background of the chart:

  {% highlight python %}
finalstacked = optionscombined.groupby(['workinghand', 'n', 'stage', 'nplayers', 'workinghicard'], as_index=False, observed=True)['workedout'].agg(['mean', 'size'])
finalstacked.sort_values('size').head(10)

finalstacked['_meansize_helper'] = finalstacked['mean'] * finalstacked['size']  #add a helper for weighted mean

stacked2 = finalstacked.groupby(['workinghand', 'n', 'stage', 'nplayers'], as_index=False, observed=True).agg(
    card_min = ('mean', 'min'),
    card_max = ('mean', 'max'),
    _helpersum = ('_meansize_helper', 'sum'),
    size = ('size', 'sum')
)

stacked2['weighted_mean'] = stacked2['_helpersum'] / stacked2['size']  #calculate weighted mean using helpers
stacked2['range'] = [f"{r['card_min']*100:.0f}-{r['card_max']:.0%}" if round(r['card_min'],2)!=round(r['card_max'],2) else f"{r['card_max']:.0%}" for _i,r in stacked2.iterrows()]
stacked2 = stacked2.drop(columns=['_helpersum'])

np.savez_compressed("finalstacked", stacked2=stacked2)  #back it uppppp
stacked2.head(10)


#generate graphs
for nplayers in range(2,11):
    tochart = stacked2[stacked2['nplayers']==nplayers]
    
    #final pivot for heatmapping
    pvt = tochart.pivot(index='workinghand', columns=['stage', 'n'], values='weighted_mean').sort_index(axis=1)
    pvtannots = tochart.pivot(index='workinghand', columns=['stage', 'n'], values='range').sort_index(axis=1)  #format the custom annotations
    
    #columns - make em pretty
    #pvt.columns = [f"{_[0].title()} ({_[1]}/5)" for _ in pvt.columns]
    pvt.columns = [f"{_[1]}/5" for _ in pvt.columns]
    pvt.index = [_.title() for _ in pvt.index]
    
    #heatmap
    fig, ax = plt.subplots(figsize=[15,5])
    sns.heatmap(pvt, ax=ax, annot=pvtannots, fmt='', annot_kws={'size':10}, cmap='RdYlGn', cbar=False, vmin=0, vmax=1, linewidths=0.5, linecolor='k')
    
    ax.set_title(f"Texas Hold 'Em Odds by Stage ({nplayers} Players)", fontsize=16)
    ax.tick_params(axis='x', labelsize=9)
    ax.tick_params(axis='y', labelsize=13)

    #borders
    _patchcolor = 'k'
    _patchweight = 3
    ax.add_patch(patches.Rectangle((0,0), 2, 6, fill=False, edgecolor=_patchcolor, linewidth=_patchweight))  #preflop
    ax.add_patch(patches.Rectangle((2,0), 4, 6, fill=False, edgecolor=_patchcolor, linewidth=_patchweight))  #flop
    ax.add_patch(patches.Rectangle((6,0), 4, 6, fill=False, edgecolor=_patchcolor, linewidth=_patchweight))  #turn
    ax.add_patch(patches.Rectangle((10,0), 5, 6, fill=False, edgecolor=_patchcolor, linewidth=_patchweight))  #river
    ax.add_patch(patches.Rectangle((0,3), 15, 3, fill=False, edgecolor=_patchcolor, linewidth=_patchweight))  #x of kind vs straight/flush

    #stage labels
    _fontsize = 13
    _dist = 6.65
    ax.text(1, _dist, "Pre-Flop", ha="center", va="center", fontsize=_fontsize)
    ax.text(4, _dist, "Flop", ha="center", va="center", fontsize=_fontsize)
    ax.text(8, _dist, "Turn", ha="center", va="center", fontsize=_fontsize)
    ax.text(12.5, _dist, "River", ha="center", va="center", fontsize=_fontsize)

    #dividers
    _lweight = 1
    _dist = 6.85
    ax.plot([0,0], [1,_dist], clip_on=False, color='k', linewidth=_lweight)
    ax.plot([2,2], [1,_dist], clip_on=False, color='k', linewidth=_lweight)
    ax.plot([6,6], [1,_dist], clip_on=False, color='k', linewidth=_lweight)
    ax.plot([10,10], [1,_dist], clip_on=False, color='k', linewidth=_lweight)
    ax.plot([15,15], [1,_dist], clip_on=False, color='k', linewidth=_lweight)

    fig.savefig(f"FinalCharts/TexasHoldEmOdds_{nplayers}Players.pdf", bbox_inches='tight')
    plt.show()
  {% endhighlight %}
</details>



And that's it!  As you may have noticed, the examples above all use the 10-player table.  The link below includes summary charts for all table sizes (2-10 players).  Note that the chart you start a hand with should be used *regardless* of how many players fold.  If 10 players were dealt in at the start of a hand, the odds that someone has a better hand than you remains 90% even if 5 players drop out (a sort of reverse-Monty Hall problem, if you will).

[Download All Summary Charts ⬇️]({{ site.baseurl }}/assets/projects/20251001_texasholdem/TexasHoldemOddsTables.zip){:download="TexasHoldemOddsTables.zip"}




## Final Thoughts
That's it!  A very fun quick little one with a couple valuable forays into under-the-hood performance management and learnings on optimizing performance with low-cost object types.  In terms of future work that could be done on this or areas I'd approach differently:

- **Statistical Shortcuts**: I alluded to this at the top, but I do still suspect there exist some elegant stats proofs that would have let me cut some corners here.  This was in some respects a bit of a ham-fisted "just do it and see what happens" approach, and I would have liked to incorporate more statistical theory to avoid having to simulate explicitly.  In the end, the rules of the game introduced enough complexities that I just couldn't get to 100% confidence I had the math right, so I played it safe.  If you know something I don't, I'd love to learn about it!  Drop me a line at Hayden.Estabrook@gmail.com.
- **Missed Data Collection Opportunity**: I realized about halfway through this that I was throwing away a lot of really good data by narrow-mindedly only logging the "my results" (i.e. player one in each game).  I could have collected 2-10x as much data from each Monte Carlo sim by modifying the logging to include all players' respective hands and results. That said, the Monte Carlo aspect was actually pretty lightweight and the benefit would be small - the real limiting steps on this analysis were memory management for large datasets and the evaluation of early-game hands.
- **Duplicate Logging/Evaluation**: Having run 9M hand simulations, a sort-of-significant portion were likely duplicates (there ~150M possible 7-card hands).  I could have maybe saved a bit of runtime by just updating these hands with a "number of occurrences" feature instead of fully re-evaluating them.
- **Edge Case Hand Rankings**:  I didn't fully incorporate *every* line in the poker rulebook into my checks.  There are technically subsequent rules around breaking ties (e.g. full houses compare hi cards on the 2-pair if tied, "kicker" cards are incorporated for hands <5 cards), and also genuine situations where players will tie and split a pot.  To keep it simple, I just modelled the first layer of tiebreaker checks and picked a random winner after that - my assumption was it would all come out in the wash.
- **Incorporating River Percentage**:  I was interested to also incorporate a dimension for "what percent of the hand we're talking about is on the river".  If you're saying you have "a pair of aces" but they're both river cards, you don't actually have any genuine advantage over any other player.  Unfortunately, due to the limitations of human eyeballs I only had two dimensions to cram data into so this one got the boot.
- **Fold Handling**: I said above that you should stick with the 10-player odds chart, even if 5 players fold.  This is a safe assumption and one I think is generally true, but I suspect in reality it would lead to underestimating your chances.  There are scenarios where a perfectly rational player would fold, without the future knowledge that an unexpected turn would have caused them to win.  In this model, no one ever folds so the true odds are probably slightly deflated.  This opens up a whole can of worms into guessing at human behaviour that I didn't want to touch, but there could be some cool future analysis around how many "effective players" are in the game based on who folded and when.

That's all for this one!  Consider my gerrymandering palette officially cleansed.

'Til next time,<br>Hayden






