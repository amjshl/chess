
# search-contempt documentation

I introduce a new form of mcts search that is a hybrid of puct search and thompson sampling, resulting in significant improvements in playing strength for odds play (a 2-player game where one side begins the game with a clear disadvantage but because of the assumed difference in playing strength, still can have significant chances to win) and also a possibility of improving compute efficiency of generating better quality training games for standard game play as well.

Briefly, puct search is an mcts algorithm that was used both to generate training games and play matches by the seminal [alpha zero](https://arxiv.org/pdf/1712.01815) alpha zero algorithm for 2-player boardgames like go, chess and shogi. This was later replicated and further developed by leela chess zero or [lc0](https://github.com/LeelaChessZero/lc0/tree/release/0.31), the open source version of alpha zero. In puct search, the visit distribution for different moves is dynamically adjusted depending on how good they are after searching certain number of nodes. The better the move is, the more it is visited/explored. In the limit of infinite nodes, this converges to the move that is the best and all the subsequent visits go towards this move.

In thompson sampling, the visit distribution from any position is fixed by a probability distribution before any exploration even begins, this does not change as more information is obtained on the position. Thompson sampling has been heavily discussed among many members of the lc0 developer community.

## Hybrid Search: combining puct and thompson sampling

In this [hybrid implementation](https://github.com/amjshl/lc0_v31_sc) of mcts, I make a modification to the pure puct search. A new integer parameter is introduced which is "search-contempt-node-limit". Here player-to-move (player 1) always uses the puct algorithm for exploring nodes with player-1-to-move till the end of search while for the opponent's (player-2-to-move) nodes, it uses the hybrid mcts search algorithm. In the hybrid algorithm I use the puct algorithm till the number of visits for that node reaches the "search-contempt-node-limit" value, at which point the visit distribution for its children are stored separately or "frozen". Once the total visits for the node exceeds that number, all subsequent visits are sampled proportional to the frozen visit distribution till search is terminated, hence preserving the frozen visit distribution. This limit is universally applied at all depth levels for the opponent's (player-2) nodes, ie search depth of 1, 3, 5 etc.

## Improvement in rating for Odds play:

This hybrid form of mcts, which I call "search-contempt" or "exploitative-exploration", has shown significant gains for queen odds, rook odds and knight odds play against weaker opponents. To understand why lets briefly go through certain unusual characteristics of odds play. Prior to 2023, even the strongest of engines, when given a handicap performed really poorly against human opponents. This is because, they would assume best play from the opponent as well. Hence it would reject all the moves that would be promising against a weaker player but would be non-optimal against a strong engine. It was thus impossible that computers could beat the top human with knight odds in long time formats. This changed when the lc0 project trained specialized neural networks on games against weaker opponents (ie engines which approximate the imperfect human play). These engines using the specialized neural networks scored better against human opponents in odds chess by adopting strategies that exploited imperfect play by humans. These strategies were imprinted in the policy and value function learned by the specialized neural network. However there is one major problem when the default mcts search algorithm is applied to these networks. The odds play gets better as more nodes are searched, but only upto a certain point, after which the search outcome or playstyle begins to resemble the strongest engines in non-odds chess which are weaker in odds chess. Hence, for the default mcts algorithm, the total nodes searched per move had to be limited in order to optimize playing strength for odds chess. Historically this was around 1000 nodes for queen odds, 10,000 nodes for rook odds and 20,000 nodes for knight odds chess.

The reason that the hybrid mcts does not suffer from this effect is that by freezing the distribution of visits for the opponent's play, we maintain the imperfect play expected of the human opponent while simultaneously allowing for higher number of visits for the side giving odds (lets say player 1), thereby increasing the strength of player 1 relative to the opponent.

Below is the results of the experiments in queen odds play (player 1 is stronger and plays without a queen and player 2 plays with the entire set of pices) for pure mcts search against a "weak" opponent, for different number of nodes searched:


Nodes searched	|	Win Percentage (%)

100		          | 23.43

800		          |	35.25

20000		        |	25.40

For the hybrid mcts search the scaling is entirely different. With "search-contempt-node-limit" set to 28

Nodes searched	|	Win Percentage (%)

100		          |	36.76

800		          |	57.81

20000		        |	81.73

It is clear from the above results that for the pure mcts search, win percentage first increases as the nodes searched increases, reaches a maximum at around 800 and then starts falling on searching even more. For the hybrid mcts search, on the other hand win percentage always increases with total number of nodes searched for fixed values of "search-contempt-node-limit". Also, in absolute terms the win percentage of the hybrid mcts search is greater than the pure mcts search for the same number of nodes searched. 

So great, search-contempt has been proven to improve strength of engines in odds chess, by virtue of more accurate modeling of weaker opponents and better scaling as the number of nodes searched increases. Surely, there is no way this will work against the top chess engines of today (which seem to play perfect chess) for playing ordinary chess. Initial evidence seems to point towards the contrary.


## search-contempt in regular chess

Inspired by the performance of search-contempt in odds chess, I ran 3 new experiments in selfplay-mode using search-contempt. This new set of experiments was different from the earlier set in a few ways. First, they started from the standard starting chess position. Second, they were between 2 players of identical strength. Third they were played by applying search-contempt from both sides, ie both the sides performed the search assuming the other was weaker, even though they are of the same strength. The details of the 3 experiments is given below

Experiment 1: 1000 nodes per move searched searched using pure mcts from both sides

Scoring Percentage : 50.00%

Drawing percentage : 92.00%

[Expt1 games](https://github.com/amjshl/chess/blob/master/games/Expt_1_selfplay_1000_nodes.pgn)

Experiment 2: 1000 nodes per move searched with hybrid mcts and search-contempt-node-limit set to 32

Scoring Percentage : 49.50%

Drawing percentage : 67.00%

[Expt2 games](https://github.com/amjshl/chess/blob/master/games/Expt_2_selfplay_1000_nodes_search-contempt_32.pgn)

Experiment 3: 32 nodes per moves searched using pure mcts

Scoring Percentage : 49.50%

Drawing percentage : 59.00%

[Expt3 games](https://github.com/amjshl/chess/blob/master/games/Expt_3_selfplay_32_nodes.pgn)

Even though, all these experiments have similar scoring percentage there are some fundamental qualitative and quantitative differences between them which have remarkable implications. Experiment 2 has much lower drawing rate compared to Experiment 1. Experiment 3 which uses 32 nodes per move without search-contempt, which is intentionally set to the same value as search-contempt-node-limit value used in Experiment 2, has an even lesser drawing rate. This is because at lower node count, the variance in the visit distribution, and hence the result (win-loss-draw) distribution is higher as the search has not had time to settle down completely yet.

Qualititively, the differences between the three experiments are striking. Experiment 1 (pure mcts with 1000 nodes), is of really high quality but is very limited in variety. Most games are similar and both sides are quick to exchange off pieces and liquidate to a drawn endgame. This resembles the quality of games that is expected of top engines performing search at 1000 nodes per move. Experiment 3 (pure mcts with 32 nodes), is of much lower quality but there is a lot more variety. There is more blundering of pieces, but occasionlly you also see interesting tactics and sacrifices. There is a huge fluctuation in the objective evaluation of the position as the game progresses. Experiment 2 is where you see the effect of search-contempt. There is an order of magnitude larger variety of positions encountered compared to both Experiment 1 and Experiment 3. The quality of the games is worse than Experiment 1 but not by much. Objective evaluation of the positions fluctuates much less than that in Experiment 3. Almost every game features interesting positions involving piece imbalances, trapped pieces, perpetual checks, long-term piece sacrifices, dynamic stalements, desperado tactics, checkmates at the middle of the board, and many more 2 or 3-move tactics compared to either Experiment 1 or 3. Almost every position has something interesting about it. For Experiment 3 and especially Experiment 1, on the other hand you have to go through numerous games to encounter interesting/imbalanced positions. Thus Experiment 2 generates much more variety with slightly less quality, while using the same compute resources as Experiment 1. Experiment 3 much less compute resources but sacrifices immensely on both quality and variety.

Why is the style of games for Experiment 2 so different? Here is an attempt to provide an intuitive explanation. The search-contempt parameter clearly alters the visit distribution favoring exploration of the moves for which the opponent misevaluates the position at lower nodes compared to higher nodes. The greater the difference in evaluation the more likely the move is to be explored. Not only does it play out unusual (from its own point of view) positions, it actively seeks out positions it is bad at evaluating or searching, which are absent or less frequent in the games generated using pure mcts.

## Why is this result significant? 

Both the qualitative and quantitative analysis of the experiments above suggests that the space of even the "reachable" chess positions explored currently, although way better than a few years ago, is minuscule compared to what can be achieved using search-contempt or any other kind of contempt in general. The search-contempt version of mcts fundamentally shifts the distribution of positions towards those that are "challenging" or "unique" thus resulting in much better coverage of the space of possible positions. The hard evidence for the fact that there still are massive holes in the set of explored positions among even the top chess engines of today (eg stockfish and leela) is the fact that both of these engines lose game pairs to each other in the chess engine competition (tcec) held a few times a year. This novel mode of generating training games attempts to fill out those holes as quickly as possible, in a way that is compute efficient.


## How to train with better compute efficiency than alpha zero?

It should be possible to train a chess engine from zero in orders of magnitude lower number of games than that used originally by alpha zero or currently used by leela chess zero which usually runs in 10s of millions. By sampling from training data that more completely covers the space of possible positions, both the rate of improvement per unit of computation, and also the ceiling for max level of engine can be significantly improved.

Here is the outline of the approximate steps involved in training such an engine. 

Step 1: Use 800 nodes per move search with search-contempt-node-limit parameter varied such that the outcome of the resulting games roughly follows the following inequality

     d/2 < w+l < 2*d ------------(1)

where by definition, 
     w + l + d = 100%

Here w, l, and d stand for the percentage of wins, losses and draws from the white perspective. As is evident from the experiments above, at high values of search-contempt-node-limit parameter (which is identical to the pure mcts engine), eg 800, the draw rate is 92% (where d/2 > w+l) and at low values of search-contempt-node-limit, eg 32, (2d < w+l). So there must be a sweet spot for the inequality above to hold true. Generate about 5000 games with those parameters and then use those set of games for training the next version of the network. 

Refer to the link below for learning rate parameters used for training the leela queen odds network. This used 100K games for a certain number of training steps using a certain learning rate schedule.

[Leela Queen Odds](https://github.com/notune/LeelaQueenOdds/blob/main/training/768x15x24h-t80_lqo.yaml)

However one major difference from the above is that the full node visit distribution information should be used for training policy unlike the queen odds network in the link above which only used the move played as the training target for policy. 

Step 2: Run one iteration of training by using the same parameters as above but run only 1/20th the number of training steps at each learning rate phase (since there are only 1/20th the number of games generated). 

Step 3: Now repeat steps 1 and 2 until improvement saturates. Note that for testing the level of the network at each stage, search-contempt can be turned off (or set to the default high value of 1000000000).

Step 4: Once improvement has saturated with current number of nodes (800 at the start), double the node count, repeat steps 1 to 3

Step 5: Keep running Step 4 until the desired level is reached or compute resources are exhausted

I strongly believe that significant improvement will be observed in level of play after around 100,000 games. Once Step 1 to 5 are complete test the network at node counts comparable to competitive engine tournaments (usually around 5 Million nodes for each move for leela) with values of search-contempt-node-limit somewhere between 3000 and 100000

I would suggest to experiment with a medium sized network initially to get quick improvements in playing strength.

I would expect the above process to result in much steeper progress and much higher saturation level than engines using pure mcts search, used to train alphazero, for the same amount of compute.

The more the compute available, the greater the size of the network can get and the higher the number of nodes searched can be.

## How to train a queen odds network from zero?

The conditions to train a network from zero in a compute-efficient manner have been motivated by those that were used to train the initial queen odds network trained to play queen odds chess. Significant improvements were observed in the network simply by playing around 200K games with a weaker opponent. 

Link to [Leela Queen Odds Training](https://github.com/notune/LeelaQueenOdds/blob/main/README.md)

However, in the following section I outline how a queen odds network can be created using search-contempt alone from zero without the need for using a different network.

Step1: From white side use 800 nodes per move search with an initial value of search-contempt-node-limit parameter, lets call it n. From the black side use n nodes per move but without search-contempt, here n < 800. Note that it is important to reset the search tree after every turn in order for the search not to interfere with each other. Run queen odds matches and set n to such a value that the same w,d,l inequality is satisfied

     d/2 < w+l < 2*d ------------(1)

Generate about 5000 games with those parameters and then use those set of games for training the next version of the network.

Step2: Run one iteration of training by using the same training parameters as that used by the queen odds network discribed above but run only 1/20th the number of training steps at each learning rate phase.

Step 3: Now repeat steps 1 and 2 until the value of n stabilizes. Note that for the case of odds play, the testing condition should be identical to training in terms of nodes per move used and value n, used for both the sides.

Step 4: Once n has stablized, double the nodes per move count from the white side and repeat steps 1 to 3. Now hopefully, the new value of n should be higher than with the initial 800 nodes per move used 

Step 5: Keep running Step 4 until the value of n converges to a satisfactory value. If the standard non-odds network at n nodes is stronger than the world champion, then likely the fuly trained queen odds network is stronger than the world champion at queen odds chess as well!


## Acknowledgements 

Credit to Noah, Naphthalin, marcus98, Hissha and many others from the lc0 discord server for the highly insightful discussions, which served as an inspiration for the search-contempt idea, its implementation and also the experimention that I ran above. Also some other users like Rust who ran independent testing of the strength improvement with search-contempt and helped point to bugs in the code. Also, credit to the lc0 developers who have painstakingly wrote, optimized, and refined the lc0 to a really high degree of performance which allowed me to run the experiments much faster than would otherwise been possible, with limited hardware.

