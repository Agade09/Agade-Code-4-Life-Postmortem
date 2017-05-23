# Agade Code 4 Life Postmortem

I will describe my minimax AI which won the Code 4 Life competition on Codingame. I will not talk too much about the evolutions of the AI but focus on the final code.

## Local referee

I wrote a [referee](https://github.com/Agade09/CG-C4L-Arena) to be able to play games locally between versions of my AI. This takes some time but it can also save time long term being able to check that versions are improving without regressions/bugs sneaking in. Some of the referee code can also be copy pasted to make a simulation based bot that looks into the future, such as a minimax.

## Choice of minimax

My first versions were heuristics with alot of if statements. My heuristics were competitive at the time but rather than improving on those and trying to keep up with the rest of the top 10, I tried early on to imagine what the meta game would be. 

The game has a relatively small branching factor, there are 4 places to go to, 5 molecules to pick from and a handful of samples to give in/take. You also have no moves to consider while you are travelling in between stations. However, I had a feeling that you would need to see quite far into the future because not much happens per turn. For example you might need 3 turns to go to the molecules and another 4 to take enough molecules to block the opponent, for a total of 7 moves per player, the equivalent of a depth 14 minimax, so the branching factor would need to be kept really low.

It was apparent that the game could be played well with heuristics, which just select 1 move. It seemed clear to me that approximations could be made in the move selection, keeping only ~a couple of promising moves, to keep the branching factor low. All these things had me feeling strongly that the best way to play C4L would be a minimax.

The first decent version of my minimax was made by copy pasting a large part of my heuristic bot into the function that generates the possible moves. So my first version was really close to my heuristic bot in a "minimax disguise", branching a bit only at DIAGNOSIS and MOLECULES in some situations. This approximation was barely better than my heuristic bot but it gave me a starting point of a minimax bot that wasn't completely crazy.

## Minimax pseudo code

If you don't know about the minimax algorithm, it is what is used in chess AIs to play at superhuman levels. It tries all your possible moves followed by all the opponent's possible moves until a certain depth is reached. The resulting positions are then given a score by an evaluation function. Typically the evaluation function is something like `Quality_of_my_position-Quality_of_his_position`. The values are backed up the tree of possible moves with you, "the maximising player", picking the move that gives the highest score and your opponent, "the minimising player", picking the move that gives the lowest score. Minimax plays assuming your enemy is as good as you. It plays to maximise the worst case in which your enemy plays perfectly. The sequence of moves of both players found by minimax is called the Principal Variation. For more details on minimax please look in places like [wikipedia](https://en.wikipedia.org/wiki/Minimax) and [chessprogramming](https://chessprogramming.wikispaces.com/Minimax). The chess programming website has information about all the improvements of the basic minimax like [alpha-beta pruning](https://chessprogramming.wikispaces.com/Alpha-Beta) and [move ordering](https://chessprogramming.wikispaces.com/Move+Ordering) which makes the alpha-beta pruning more efficient.

In chess moves are sequential but in this game moves are simultaneous. I decided to minimax as if the game was sequential in the sense that my opponent picks the move that best counters my move, even though he couldn't be sure I would make that move. This might be overly pessimistic but I felt like it was in the "maximise the worst case" spirit of minimax.

In pseudo code this was my minimax function with alpha beta pruning:
```
struct variation{
    double score;
    vector<array<action,2>> Moves;
};

variation Minimax(state,depth,max_depth,alpha,beta){
    if(depth==max_depth){
        return variation{Eval(state),{}};
    }
    array<vector<action>,2> Branch={Possible_Moves(state,player0),Possible_Moves(state,player1)};
    variation Best_var={-infinity,{}};
    for(action mv:Branch[0]){
        variation Best_var2{+infinity,{}};
        local_beta=beta;
        for(action mv2:Branch[1]){
            state2=state;
            SimulateActions(state2,{mv,mv2});
            variation var=Minimax(state2,depth+1,max_depth,alpha,local_beta);
            if(var.score<Best_var2.score){
                Best_var2=var;
                Best_var2.Moves.insert(at beginning,{mv,mv2});
            }
            local_beta=min(var.score,local_beta);
            if(local_beta<=alpha){
                break;
            }
        }
        if(Best_var2.score>Best_var.score){
            Best_var=Best_var2;
        }
        alpha=max(alpha,Best_var2.score);
        if(beta<=alpha){
            break;
        }
    }
    return Best_var;
}
```

I can then make a move by calling `Minimax(max_depth)` and outputting the first move of player 0 in the principal variation. Since I can't know what depth I will be able to search at in 50ms given the current state, I use [iterative deepening](https://chessprogramming.wikispaces.com/Iterative+Deepening). Iterative deepening involves searching at depth 1,2,3... until you run out of time and using the result of the last completed search. So if you run out of time at depth 6 you use the result of the depth 5 search. In pseudo code:

```
action Decide_Move(state){
    variation best_var;
    depth=1;
    while(depth<=201-turn){//Don't look past end of game
        try{
            best_var=Minimax(S,0,depth,-inf,+inf);//Minimax throws an exception if time runs out
            ++depth;
        }
        catch(...){
            break;
        }
    }
    return best_var.Moves[0][0];
}
```

An easy improvement on the above alpha-beta pruned minimax is to order moves according to the principal variation found at the previous depth. Indeed, alpha beta pruning works best if the best moves are tested first and the best moves are often the same at depth N and N+1.

A good place to practice minimax on Codingame is the [Tron game](https://www.codingame.com/multiplayer/bot-programming/tron-battle) with the "[Voronoi heuristic](https://kootenpv.github.io/2016-09-07-ai-challenge-in-78-lines)" as your evaluation function.

Now that I've talked about the basics of minimax which many of you know about I need to talk about the two important parts of my code:
* The `Possible_Moves(state,id)` function
* The `Eval(state)` function

## Simulation

To be able to minimax you need to be able to simulate the game state into the future assuming players made certain moves. For this you need to reimplement the rules of the game. But you have to decide what to do with diagnosing samples because you don't know what the sample's score, cost and expertise is going to be because undiagnosed samples are unknown to the player. In my simulations when I diagnose a sample I give it a score of 0 and handle these diagnosed unknown samples differently.

## Sorting samples

In several parts of the code it was important to sort samples such as:
* When deciding which sample to complete first
* What to get from the cloud
* What to throw in the cloud

For this task I scored sample by:
```
(MoleculesAvailable && EnoughSpaceToTake ? 100 : MoleculesAvailable? 1 : 0)-MissingMolecules*1e-3
```

So I sort in order of producibility and discriminate among equally producible samples by the number of missing molecules. This seems pretty dumb, it doesn't even take into account the score of the sample, but I felt like I had more important things to work on during the contest and it seemed clear to me that producibility was the most important thing, getting blocked by the enemy is the worst thing that can happen.

## Evaluation

I evaluate a position as `Eval(state)=Eval(state,player 0)-Eval(state,player 1)`, the difference between the quality of my position and my opponent's position. I now describe my `Eval(state,id)` function.

If the state is at turn 201 then return the score of the player. Nothing else matters at the end of the game. Otherwise the evaluation is given by:

* `ExpertiseCoeff*player.expertise`
* `player.score`
* `0.15*(MinScore[rank]+ExpertiseCoeff)` for an undiagnosed sample
* `0.175*(MinScore[rank]+ExpertiseCoeff)` for a diagnosed unknown sample
* `0.05*(sample.score+ExpertiseCoeff)` for an "unproducible" sample whose needed molecules aren't available to the player
* `0.5*(sample.score+ExpertiseCoeff)` for a "producible" sample whose needed molecules are available to the player
* `0.85*(sample.score+ExpertiseCoeff)` for a sample which is ready to be produced at the LABORATORY.
* `ExpertiseCoeff=10` serves to smooth the value a bit. A score 10 sample shouldn't be valued 10 times more than a score 1 sample. The expertise is valuable and the 10 to 1 ratio becomes 21 to 11 which seems more reasonable.
* If I have producible samples but don't have enough space to take the molecules to complete any of those, then they are all considered unproducible
* Sort the samples and give a `1e-2*pow(0.5,i)` penalty for each missing molecule to complete sample i. This gives a small incentive to collect molecules for the samples, in order, but the hope is to have enough depth to see a sample made ready and get the `0.85*(sample.score+ExpertiseCoeff)`.
* `(1e-2/4)*player.space` (the maximum 10 molecule carrying space). This is to avoid the AI filling up with molecules without purpose.
* If I'm carrying an undiagnosed sample, give a tiny `0.01*distance_to_DIAGNOSIS` penalty. The distance being your `player.eta + Distance[player.location][DIAGNOSIS]`. This is minor but it was to stop my AI from waiting in some situations where the depth of the search took it to a point where it was travelling in between SAMPLES and DIAGNOSIS. Without encouraging it to get closer to DIAGNOSIS it saw no difference between waiting a turn or not.

`MinScore[rank]` is the minimum score for that rank. 1,10,30 respectively for the 3 different ranks.

When checking if samples are ready to be produced and computing the missing molecules you have to take into account the molecules that are reserved for other samples. Make sure you're not counting the same A molecule for 2 samples. Count it only for the first sample in the sorted order.

I didn't count future expertise gain to decrease the cost of other samples, there might be a possible improvement there.

As you can see, in my evaluation of the state a sample is worth `sample.score+ExpertiseCoeff` and you get more of that score as the sample advances in completion. Handing in a sample at the LABORATORY only gives you the last 15% of the value of that sample so the IA isn't too desperate to go hand in samples right away if it can make another sample go from "producible" to "ready", gaining 35% of the value of that other sample.

A sample loses alot of its value when it becomes "unproducible", putting it at a lower % of its value than if it was undiagnosed. This puts "blocking" into my evaluation. I will try to block the enemy while expecting the enemy to do the same.

## Move Generation

This was a very important part of the AI. In a normal minimax you would just look at all legal moves but as I explained I thought it would be important to restrict the search for extra depth but also to avoid crazy behaviours the AI had as I was developing it. Even the final version in the arena wastes turns from time to time, for example by putting a sample in and out of the cloud because it sometimes sees no disadvantage to "doing it later" depending on the specific situation and the depth reached in the search.

My `Possible_Moves(state,id)` function is split into 4 parts for the 4 different locations of the player.

If the player is at the starting position the only possible moves is to go to SAMPLES. If the player's eta is >0 then the only move is to WAIT. The possibles moves at the 4 different locations are described below.

### Samples

* If carrying less than 3 samples take a sample of rank 1 if total expertise is < 9, rank 2 if total expertise is < 12 and rank 3 otherwise.
* If carrying 3 samples go to DIAGNOSIS

So only 1 possible move at SAMPLES, taking undiagnosed samples or going to DIAGNOSIS.

### Diagnosis

* If the player has an undiagnosed sample the *only* move is to diagnose it, I didn't think there would be many situations where any other move would be better.
* If the player has 3 samples, a possible move is to go to MOLECULES.
* If the player has less than 3 samples and there are samples in the cloud a possible move is to grab the best one according to my sorting of samples.
* If the player has samples a possible move is for the player to drop his worst sample according to my sorting of samples.
* If the player has less than 3 samples, a possible move is to go to SAMPLES.
* If the player has a sample which is ready for production a possible move is to go to the LABORATORY.

I had, and still have issues at the DIAGNOSIS station where my AI would swap samples in and out and waste time for many different reasons. But one situation that was happening alot was my AI waiting at DIAGNOSIS while the enemy was waiting at the lab. I didn't manage to get rid of this behaviour in a clean way so I improved the situation alot by restricting the AI's moves by adding:

* If the player has no producible samples on him or in the cloud and `enemy_score+score_of_enemy_produceable_samples>=current_score` then the player can only throw away its samples if it has any or go to the SAMPLES station otherwise.

This hack was worth ~8% win rate against the previous version. This problem was costing alot of games.

### Molecules

* If the player has space and a molecule type is available a possible move is to take one of those molecules.
```
if(sample is ready){
    Possible.Add(GOTO LABORATORY);
}
else if(player.samples.size() < 3){
    Possible.Add(GOTO SAMPLES);
}
else{
    Possible.Add(GOTO DIAGNOSIS);
}
```
* If the enemy isn't at SAMPLES and `player.score + score_of_ready_samples > enemy.score + score_of_enemy_produceable_samples` then a possible move is to wait at the MOLECULES.

There are probably a decent amount of situations where it's good to go to DIAGNOSIS to take samples from the cloud which my AI won't do unless it has 3 samples. I didn't get around to trying to allow it to go to DIAGNOSIS from MOLECULES in more situations.

### Laboratory

* If a sample is ready for production a possible move is to produce one of those samples. I don't consider producing in a different order than however the samples come.
* If a sample is ready, the enemy isn't at SAMPLES and I'm currently winning by the same criteria as in MOLECULES then a possible move is to wait at LABORATORY instead of handing the sample right away.
* If no samples are ready possible moves are going to MOLECULES and SAMPLES.
* If no samples are ready and there are producible samples in the cloud and enough samples in the cloud in total so that I can fill up to 3 then a possible move is also to go to DIAGNOSIS

## Performance

I had between ~25-50k calls to Simulate() per turn and reached a depth which varied greatly between 4 (moves per player) when both players were at the molecules and ~15. Most of the compute time was in the evaluation function. To improve depth/performance I think the most important next step would have been to find a cheap hashing function for the game state to be able to make [transposition tables](https://chessprogramming.wikispaces.com/Transposition+Table) to store values of `Eval(state)` and best moves for ordering.

## Details

* When the referee tells you there are 0 of a molecule, there might actually be -1 if both players took the last one at the same time. To get the proper state you should make sure `Avaiable['A']=5-Player0.Molecules['A']-Player1.Molecules['A']` 

## Conclusion

I finished with ~630 lines of C++. I'm a bit surprised I managed to win by such a large margin without being contested by other minimaxes. From reading other people's postmortems it seems like many tried and gave up on the idea. I personally had a strong feeling that it was doable and that it would be the meta. I struggled with it Sunday afternoon but got my first version with >50% win rate (against my heuristics) on Sunday evening. It made a ton of weird moves though so I didn't submit a minimax until maybe Monday. The first minimaxes I had were looking at a limited depth because the AI often went crazy with too much depth. I believe it was Thursday/Friday that I managed to fix enough issues to submit my first 40ms version.

It was harder than I'd hoped to get a minimax to work. I hadn't anticipated that it would do so many strange moves like putting samples in and out of the cloud and going back and forth between stations. In the end the bot is a somewhat fragile equilibrium between restricting the possible moves, evaluating positions well enough and getting sufficient depth to see future rewards. I think the key was to start with a dumbed down minimax with severely restricted moves and build up from there.

If it comes out as a multiplayer I think surely other people would beat my AI. However I do think this type of minimax bot must be the meta for this game.

It feels a bit unreal having finished 1st, 2nd and 1st these last three contests.