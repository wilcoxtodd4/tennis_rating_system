# tennis_rating_system
TRS - Todd's Rating System (aka Tennis Rating System) is a personally developed system for rating tennis players based on results.  The system contains one rating methodology for singles and two for doubles (one based solely on individual performance and one with more dependency on common partnerships).

The primary application has been high school tennis results in Oregon (I am a high school coach).  In our 2025 district tournament, it's predictions has accuracy rates of 100% in singles and 94% in doubles.

This project is primarily used as a way to train myself in Python.  Parts of the modules were built at various stages of my training, so it includes a variety of different styles and includes a few things where I've resorted to "it's good enough for now".  Most of it has been moved into jupyter notebooks

Python Libraries Used:
  pandas
  numpy
  matplotlib
  seaborn
  sqlite3
  json
  csv
  pprint (development only)
  requests
  urllib.request, urllib.parse, urllib.error
  

The basic flow of the data and rating system is:
  -- Gather a list of teams/schools and store as a csv file (only updated annually)
  -- Gather a list of players - use python to read in the json file to create a csv (weekly).  There are 3 player lists (full state, our conference, and teams involved in our biggest tournament)
  -- Connect to the api of tennisreporting.com to gather all the match results for all the teams in the state.  Each school's matches will be in a separate json file.
  -- Extract and clean the match scores from the json files.  This data is dirty.  The GUI interface to enter match scores has no input filters or data checks, so a lot of garbage goes in.  Some issues include:
    -- Bad name formatting, especially with hyphenated last names
    -- Incorrect player names
    -- Backwards scores (scores reflecting that the losing team won)
    -- Impossible scores (scores like 61-0 in a set that was probably 6-1 are common)
    -- Tie-break scores being listed as set scores
  -- Create the ratings for singles and doubles for each player

Assumptions for Ratings:
Each players rating starts with an assigned rating based on their position on the team.
These assumptions help differentiate players early in the season when not many matches have been played.  The have much less impact late in the season.
  -- By rules, players must be played in order of strength (e.g. #1 can beat #2).
  -- The level difference in high school varsity tennis is massive.  They range from D1 prospects down to literal beginners without much athletic training.
  -- The 2025 rating assumptions are below (based on 2024 actuals):
S1,27
S2,23.4
S3,21
S4,19.5

D1,23.2
D2,20.1
D3,18.5
D4,17
D5,15.3

What do the ratings mean?
The ratings make sense mostly when comparing players.  For players matched up against each other, one rating point gap equates to 1 expected break of serve per match.
  -- For non-tennis people, opponents take turns serving one game each.  The server is supposed to win their service game.  If the returning player wins, that is a break of serve.
  -- Another way to look at it is breaking the match up into two game blocks (I serve, you serve).  A break of serve means I won both games instead of us splitting them 1-1.
  -- Some typical tennis scores (and their break count) are:
    -- 6-0, 6-0 (6 breaks of serve - I won all my games and also won your game 6 times)
    -- 6-4, 6-2 (3 breaks of serve - I won my game and yours 3 times more than you won them both)
The rating gap between two players mimics those breaks of serve, and can act as a predictor of both the winner and the score.
  A player rated 27 versus a player rated 25 is expected to win by 2 breaks of serve - approximately 6-4, 6-4.
    -- The same for a player rated 16 vs 14.

Looking back at the assumptions, you can see quite a range with #1 singles players expected to have average ratings of 27, while 5th doubles players have expected average ratings of 15.3.
  -- That 12-point gap represents that, on average, the #1 singles (27) can beat the #3 singles (21) 6-0, 6-0, and the #3 singles player is probably 6-0, 6-0 better than a 5th doubles player.

Ratings in reality:
  -- The top rated player for the year ended up rated 38.8
  -- The lowest rated player for the year ended up rated 4.1

Rating Methodology (Conceptual):
  1) The process calculates the difference between expected results and actual results for each match.
  2) It then assigns a match-level dynamic rating to each player in the match based on the result and the players existing rating.
     a) If a 24 beats a 20 by two breaks of serve, the dynamic ratings for that match would be 23 and 21 (they still average 22 as a pair but the dynamic gap is 2 instead of the expected 4).
  3) It does this for every match for every player on the season.  Then it finds each players' average dynamic rating for all matches played.
  4) It iterates through this process as many times as requested, using the average dynamic rating as a new input for calculating the ratings for the new iteration.
     a) Perhaps the aforementioned 24 vs 20 both did well in many other matches and now come in as 26 and 23.  The new iteration calculates dynamic ratings of 25.5 and 23.5.
  5)  Over several iterations (10 is frequently enough), the ratings stabilize and stop shifting more than 0.01 points per iteration.

Known Issues:
  -- The accuracy of the ratings, especially at the top, depend on chances to play against other top players.  Unfortunately, that doesn't always happen.
    -- To achieve a rating of 38.8, you'd need a score of 6-0, 6-0 against a player rated 32.8 (or 6-3, 6-3 vs a player rated 35.8)
      -- There are only 16 players in Oregon rated at 32.8 or above.  With different school sizes and different leagues, that doesn't always happen.

  -- Doubles Ratings are difficult to get right.  The system can only tell how two players did against two other players, not which specific players did well or poorly.
    -- Two rating methods exist for doubles due to this:
      -- The INDV method adjusts all players equally based on match results:
        -- A 22 and a 20 (average 21) vs an 18 and 16 (average 17) are expected to win by 4 breaks.  If they win by 2 instead, the dynamic ratings are 21/19 (average 20) and 19/17 (average 18).
      -- The TEAM method treats team as an entity and calculates how good that team is against other teams.
        -- After that, it creates a rating for each player based on how good they are with that partner compared to how good they and their partner are with other partners.
          -- Players A/B have a rating of 24 together.
            -- But Players A/C are rated 25, A/D are rated 27.
            -- And Players B/E are rated 23 while B/F are rated 21.
            -- So it seems like maybe A is the stronger part of the A/B team.  So it assigns A a rating of 26 and B a rating of 22.  They still average 24.
    -- Those systems are very different and can vary by up to 10 points, but most are within 2 points.
    -- The issue is that some partners will get repeatedly iterated away from each other - one going up every iteration and one going down.
      -- To put it in English, the process can become more stable across all the matches by moving one partners rating up while the other goes down.  Their average stays the same, but they diverge beyond being reasonable.
        -- This can happen when one player has a very bad result or two with different partners while their partner has a very good result or two with different partners.
        -- It assumes that the player going down is responsible for every bad result while the good player is responsible for every good result.
        -- Limiting the total iterations prevents this from spiralling.
          
    
