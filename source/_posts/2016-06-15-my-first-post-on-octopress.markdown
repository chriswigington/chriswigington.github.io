---
layout: post
title: "Symbols as Methods, or Accidental Metaprogramming in Tic-Tac-Toe"
date: 2016-06-15 18:30:53 -0400
comments: true
categories: "Flatiron School"
---
This is a short account of how I ended up stumbling into using Ruby's #send method in order to provide a solution to a coding issue.

After getting the basic game mechanics worked out for our ruby implementation of Tic-Tac-Toe, I thought it would be fun to see if I could make the computer play a little more intelligently. I turned to the trusty [Wikipedia article on Tic-Tac-Toe](https://en.wikipedia.org/wiki/Tic-tac-toe#Strategy) for reference, which provided an eight step strategy for the game:
<!-- more -->
1. Take a winning move.
2. Block an opponent's winning move.
3. Create a fork.
4. Block an opponent's fork.
5. Take the center square.
6. Take the opposite corner of your opponent.
7. Take an empty corner.
8. Take an empty side.

Each one of these steps was implemented as a method which was fed the state of the board, then returned either a valid position to choose if one was available, or the value nil if there were none. Here's an example of what the first step in this strategy looked like prior to refactoring:

``` ruby
def winning_move(board)
  # set position initially to nil
  position = nil
  # iterate through the winning combinations
  Board::WIN_COMBINATIONS.each do |combo|
    mark_count = 0
    nil_count = 0
    nil_spot = nil
    # iterate through the values of the specific combination
    combo.each do |spot|
      if board.grid[spot] == player
        mark_count += 1
      elsif board.grid[spot] == nil
        nil_count += 1
        nil_spot = spot
      end
    end
    # if you have two marks in a row and the other spot is empty
    if mark_count == 2 && nil_count == 1
      # assign the winning move to position
      position = nil_spot
    end
  end
  # method will return the winning move or nil
  position
end
``` 

If there were no available moves at that particular step, we would want to move on to the next. Implementing these steps was interesting in and of itself (not to mention the process of refactoring them...), but the logic for the decision-making process led to some particularly interesting issues. My first version was a series of basic if and elsif statements, like so:

``` ruby
def make_smart_move
  if winning_move(board)
    winning_move(board)
  elsif block_winning_move(board)
    block_winning_move(board)
  elsif fork_move(board)
    fork_move(board)
  elsif block_fork_move(board)
    block_fork_move(board)
  elsif empty_center(board)
    empty_center(board)
  elsif opposite_corner(board)
    opposite_corner(board)
  elsif empty_corner(board)
    empty_corner(board)
  elsif empty_side(board)
    empty_side(board)
  end
end
```

This relies on the conditional statements seeing whether our method returns a value or nil, and then running the method again for the return value if one was available. If repeating the same code over and over again and having methods over five lines are both anathema to good coding, this obviously was not a preferable solution to creating the logic for this decision-making process. 

Racking my brain for a solution to this problem, I arrived at a question: Is it possible to store methods in a data collection such as an array? My first instinct was to see what would happen if you just attempted to initiate an array filled with methods:

``` ruby
array = [
  winning_move(board),
  block_winning_move(board),
  fork_move(board),
  block_fork_move(board),
  empty_center(board),
  opposite_corner(board),
  empty_corner(board),
  empty_side(board)
]
```

While this might initially seem as if it would be an array of methods, what this actually creates is an array of the methods' return values. So, if the computer were playing the first move, for example, our array would return this:

``` ruby
 => [nil, nil, nil, nil, 4, nil, 0, 1]
```
This actually would work fine for providing our computer player's decision logic, as you could just iterate through the array until you find a non-nil value and return that as the position to play. This also has the added advantage that each method would only be run once. 

However, my curiosity had been piqued as to whether it's actually possible to store methods in an array, and not just their return values, which led me eventually to this (dubious) solution:

``` ruby
# create a variable to hold the list of methods as symbols
@move_methods = [:winning_move, :block_win, :fork_move, :block_fork, 
  :empty_center, :opposite_corner, :empty_corner, :empty_side]

# Find the move that returns a value
def find_correct_method(board)
  move_methods.find do |method|
    send(method, board)
  end
end

# Return the position that works
def make_smart_move
  send(find_correct_method, board)
end
```

While this solution relies on running the methods twice, as in the long conditional statement, it does at least result in much shorter code per method. It also relies heavily on an Object class method called [#send](http://ruby-doc.org/core-2.3.1/Object.html#method-i-send) in order to take the symbols and actually run them as methods.

What #send does is take in a first parameter of a symbol or string, which it takes to be the method, and then runs the method while sending any further parameters to the method as arguments. As I investigated more into #send and other similarly useful methods like [#respond_to?](http://ruby-doc.org/core-2.3.1/Object.html#method-i-respond_to-3F), most of the resources I ran across spoke of them in reference to metaprogramming, a topic we had been very *gravely* warned about.

So if nothing else, this anecdote provides a good example (warning?) of how Ruby allows us to solve problems in a variety of different ways, including ones that maybe are not very advisable. In hindsight I would have been better off just returning the values of the methods to the array as a solution, but I'm very interested to hear what others did if they ran into similar issues.

Chris