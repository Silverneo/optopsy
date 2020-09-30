[![Downloads](https://pepy.tech/badge/optopsy)](https://pepy.tech/project/optopsy)
[![CircleCI](https://circleci.com/gh/michaelchu/optopsy.svg?style=shield)](https://circleci.com/gh/michaelchu/optopsy)
[![Test Coverage](https://api.codeclimate.com/v1/badges/37b11e992a6900d30310/test_coverage)](https://codeclimate.com/github/michaelchu/optopsy/test_coverage)
[![Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/ambv/black)

# Optopsy

Optopsy is a nimble backtesting and statistics library for option strategies, it is designed to answer questions like
"How do vertical spreads perform on the SPX?" or "Which strikes and/or expiration dates should I choose to make the most potential profit?"

**This library is currently being rewritten and is in beta state, use at your own risk**

## Supported Option Strategies
* Calls/Puts
* Straddles/Strangles (WIP)
* Butterflies/Iron Condors (WIP)
* Many more to follow

## Methodology
Most backtesting tools for option strategies don't reveal how the options are backtested, since this is open sourced, I
will try to be as transparent as possible on how the algorithm works. You can decide if it works for your situation. My aim
for this libary is not to be able to backtest every possible event driven scenarios out there, but to focus on answering the core questions
laid out in the introduction. There are no fancy features such as being 'event driven', but theoretically, it is possible by injecting
a list of entry and exit dates created from external sources based on indicators.
 
There are two main parts of the library:

* Statistics Module - Focused on generating statistics on various options strategies (Independent of chronological events)
* Backtester Module (Coming Soon) - Replay the generated statistics with chronological order to create an **approximated** backtest.

The algorithm for the statistics module is as follows:

1. Evaluate all option chains provided (filter for desired the entry and exit dates and calculate the profit/loss of each 
individual option chain)
2. Group each 'evaluated' option chain into buckets (buckets are assigned to 'Days to Expiration' (grouped by days in intervals of 7, by default), 
and either 'delta' or 'strike distance percent' from current price (grouped in intervals of 5%, by default). Obviously, the smaller the intervals, the more accurate the results should be
3. Aggregate all the option chains into their buckets and calculate the average profit loss for each bucket combination
4. Construct the legs of the option strategy with the previous aggregate amounts and net out the profit/loss
5. The result will contain approximated profit/loss amounts (and other statistics such as min/max, distributions) for all the combinations of inputs (strike dist %/ DTE) of the option strategy 

Obviously this is a close approximation of a backtest (due to the bucketing and averaging) and also it does not take into account
real world events in a chronological order. For that there, is the backtester module that will replay each backtest ordered
by expiration dates with a running balance to simulate what would have happened.

### Notes
As the algorithm is **heavily** based on bucketing and approximations to improve performance, it is not recommeded to 
make trade decisions based on the results solely from this library. Please use at your own risk.

## Usage

### Use Your Data
* Use data from any source, just provide a Pandas dataframe with the required columns when calling optopsy functions.

### Dependencies
You will need Python 3.6 or newer and Pandas 0.23.1 or newer and Numpy 1.14.3 or newer.

### Installation
```
pip install optopsy==2.0.0b1
```

### Example

Let's see how long calls perform on the SPX on a small demo dataset on the SPX:
Download the following data sample from DeltaNeutral: http://www.deltaneutral.com/files/Sample_SPX_20151001_to_20151030.csv

This dataset is for the month of October in 2015, lets load it into Optopsy. First create a small helper function
that returns a file path to our file. We will store it under a folder named 'data', in the same directory as the working python file.
```
def filepath():
    curr_file = os.path.abspath(os.path.dirname(__file__))
    return os.path.join(curr_file, "./data/SPX_20151001_to_20151030.csv")
```

Next lets use this function to pass in the file path string into Optopsy's `csv_data()` function, we will map the column
indices using the defined function parameters. We are omitting the `start_date` and `end_date` parameters in this call because
we want to include the entire dataset. The numeric values represent the column number as found in the sample file, the
numbers are 0-indexed:
```
import optopsy as op

spx_data = op.csv_data(
    filepath(),
    underlying_symbol=0,
    underlying_price=1,
    option_type=5,
    expiration=6,
    quote_date=7,
    strike=8,
    bid=10,
    ask=11,
)
```  
The `csv_data()` function is a convenience function. Under the hood it uses Panda's `read_csv()` function to do the import.
There are other parameters that can help with loading the csv data, consult the code/future documentation to see how to use them.

Optopsy is a small simple library that offloads the heavy work of backtesting option strategies, the API is designed to be simple
and easy to implement into your regular Panda's data analysis workflow. As such, we just need to call the `single_calls()` function
to have Optopsy generate all combinations of a simple long call strategy for the specified time period.

```
long_single_calls = op.singles_calls(spx_data, strike_dist_pct_interval=0.05, side="long")
```

At this point, we will be returned a Pandas DataFrame containing the statistics of running long calls on the SPX (**The example below is truncated**):

|     | dte_range   | strike_otm_pct_range   |   long_profit_pct_min |   long_profit_pct_max |   long_profit_pct_mean |   long_profit_pct_median |   long_profit_pct_std |   long_profit_pct_count |
|-----|-------------|------------------------|-----------------------|-----------------------|------------------------|--------------------------|-----------------------|-------------------------|
|   0 | (0, 7]      | (-1.0, -0.95]          |                 -0.01 |                  0.07 |                  0.019 |                     0.02 |             0.0191669 |                      20 |
|   1 | (0, 7]      | (-0.95, -0.9]          |                 -0.01 |                  0.06 |              0.0242857 |                     0.02 |             0.0201424 |                      21 |
|   2 | (0, 7]      | (-0.9, -0.85]          |                     0 |                  0.07 |                 0.0265 |                     0.02 |             0.0192696 |                      20 |
|   3 | (0, 7]      | (-0.85, -0.8]          |                 -0.01 |                  0.06 |              0.0211111 |                     0.02 |             0.0198714 |                      27 |
|   4 | (0, 7]      | (-0.8, -0.75]          |                     0 |                  0.08 |              0.0288889 |                     0.02 |             0.0222457 |                      27 |
|   5 | (0, 7]      | (-0.75, -0.7]          |                 -0.01 |                  0.08 |              0.0251515 |                     0.02 |             0.0227927 |                      33 |
|   6 | (0, 7]      | (-0.7, -0.65]          |                 -0.01 |                  0.09 |                  0.031 |                    0.025 |             0.0250947 |                      40 |
|   7 | (0, 7]      | (-0.65, -0.6]          |                 -0.01 |                  0.09 |              0.0256757 |                     0.02 |             0.0253356 |                      37 |
|   8 | (0, 7]      | (-0.6, -0.55]          |                 -0.02 |                  0.09 |              0.0252055 |                     0.02 |             0.0272901 |                      73 |
|   9 | (0, 7]      | (-0.55, -0.5]          |                 -0.02 |                   0.1 |              0.0291772 |                     0.03 |             0.0255877 |                     158 |
|  10 | (0, 7]      | (-0.5, -0.45]          |                 -0.02 |                  0.11 |               0.032337 |                     0.03 |             0.0293737 |                     184 |
|  11 | (0, 7]      | (-0.45, -0.4]          |                 -0.02 |                  0.12 |              0.0357711 |                     0.03 |             0.0321018 |                     201 |
|  12 | (0, 7]      | (-0.4, -0.35]          |                 -0.02 |                  0.13 |              0.0392308 |                     0.04 |             0.0351624 |                     247 |
|  13 | (0, 7]      | (-0.35, -0.3]          |                 -0.02 |                  0.15 |              0.0455743 |                     0.04 |             0.0389481 |                     296 |
|  14 | (0, 7]      | (-0.3, -0.25]          |                 -0.03 |                  0.17 |              0.0539514 |                     0.05 |              0.045708 |                     329 |
|  15 | (0, 7]      | (-0.25, -0.2]          |                 -0.03 |                   0.2 |              0.0617045 |                     0.05 |             0.0539594 |                     352 |
|  16 | (0, 7]      | (-0.2, -0.15]          |                 -0.04 |                  0.26 |              0.0779634 |                     0.07 |             0.0673121 |                     383 |
|  17 | (0, 7]      | (-0.15, -0.1]          |                 -0.06 |                  0.37 |               0.105468 |                     0.09 |             0.0919648 |                     417 |
|  18 | (0, 7]      | (-0.1, -0.05]          |                 -0.12 |                  0.69 |               0.176399 |                     0.15 |              0.158725 |                     461 |
|  19 | (0, 7]      | (-0.05, 0.0]           |                    -1 |                  7.62 |                0.63998 |                     0.37 |                1.0276 |                     505 |
|  20 | (0, 7]      | (0.0, 0.05]            |                    -1 |                    68 |                2.33606 |                    -0.89 |               8.64909 |                     269 |
|  21 | (0, 7]      | (0.05, 0.1]            |                    -1 |                    -1 |                     -1 |                       -1 |                     0 |                       2 |
|  22 | (7, 14]     | (-1.0, -0.95]          |                  0.01 |                  0.09 |              0.0483333 |                    0.045 |             0.0272475 |                      12 |
|  23 | (7, 14]     | (-0.95, -0.9]          |                  0.01 |                   0.1 |              0.0545455 |                     0.05 |             0.0230377 |                      22 |
|  24 | (7, 14]     | (-0.9, -0.85]          |                  0.01 |                  0.09 |               0.048125 |                     0.05 |             0.0210456 |                      16 |
|  25 | (7, 14]     | (-0.85, -0.8]          |                  0.01 |                  0.11 |                  0.062 |                     0.06 |             0.0252566 |                      20 |
|  26 | (7, 14]     | (-0.8, -0.75]          |                  0.01 |                  0.11 |                 0.0584 |                     0.06 |             0.0271846 |                      25 |
|  27 | (7, 14]     | (-0.75, -0.7]          |                  0.01 |                  0.12 |              0.0642857 |                     0.06 |              0.029761 |                      21 |
|  28 | (7, 14]     | (-0.7, -0.65]          |                  0.02 |                  0.12 |              0.0678788 |                     0.06 |             0.0244639 |                      33 |
|  29 | (7, 14]     | (-0.65, -0.6]          |                  0.02 |                  0.13 |              0.0677778 |                     0.07 |             0.0327383 |                      27 |
|  30 | (7, 14]     | (-0.6, -0.55]          |                  0.02 |                  0.14 |              0.0691111 |                     0.07 |             0.0287483 |                      45 |
|  31 | (7, 14]     | (-0.55, -0.5]          |                  0.02 |                  0.15 |               0.069386 |                     0.07 |             0.0299641 |                     114 |
|  32 | (7, 14]     | (-0.5, -0.45]          |                  0.02 |                  0.16 |               0.080719 |                     0.08 |             0.0334645 |                     153 |
|  33 | (7, 14]     | (-0.45, -0.4]          |                  0.02 |                  0.17 |              0.0868072 |                     0.08 |             0.0383135 |                     166 |
|  34 | (7, 14]     | (-0.4, -0.35]          |                  0.02 |                  0.19 |              0.0940102 |                     0.09 |              0.041301 |                     197 |
|  35 | (7, 14]     | (-0.35, -0.3]          |                  0.02 |                  0.21 |               0.108085 |                      0.1 |             0.0436594 |                     235 |
|  36 | (7, 14]     | (-0.3, -0.25]          |                  0.03 |                  0.25 |               0.125094 |                     0.12 |             0.0523545 |                     265 |
|  37 | (7, 14]     | (-0.25, -0.2]          |                  0.03 |                   0.3 |               0.147393 |                     0.14 |             0.0619861 |                     280 |
|  38 | (7, 14]     | (-0.2, -0.15]          |                  0.04 |                  0.38 |               0.182899 |                     0.18 |             0.0776728 |                     307 |
|  39 | (7, 14]     | (-0.15, -0.1]          |                  0.05 |                  0.54 |               0.245663 |                     0.24 |              0.105847 |                     332 |
|  40 | (7, 14]     | (-0.1, -0.05]          |                  0.07 |                  0.97 |               0.401595 |                     0.39 |               0.18451 |                     370 |
|  41 | (7, 14]     | (-0.05, 0.0]           |                 -0.46 |                   4.4 |                1.01542 |                    0.865 |              0.676838 |                     404 |
|  42 | (7, 14]     | (0.0, 0.05]            |                    -1 |                    32 |                 1.5167 |                    -0.73 |               4.44642 |                     388 |
|  43 | (7, 14]     | (0.05, 0.1]            |                    -1 |                 -0.83 |              -0.928333 |                   -0.935 |             0.0629512 |                      36 |

Since this is just a regular Pandas DataFrame, let's rewrite the previous function with a Panda's round function: 

```
long_single_calls = op.singles_calls(spx_data, strike_dist_pct_interval=0.05, side="long").round(2)
```

This results in a cleaner output, at this point you may use regular Panda's function to further analyze the results as you need

|     | dte_range   | strike_otm_pct_range   |   long_profit_pct_min |   long_profit_pct_max |   long_profit_pct_mean |   long_profit_pct_median |   long_profit_pct_std |   long_profit_pct_count |
|-----|-------------|------------------------|-----------------------|-----------------------|------------------------|--------------------------|-----------------------|-------------------------|
|   0 | (0, 7]      | (-1.0, -0.95]          |                 -0.01 |                  0.07 |                   0.02 |                     0.02 |                  0.02 |                      20 |
|   1 | (0, 7]      | (-0.95, -0.9]          |                 -0.01 |                  0.06 |                   0.02 |                     0.02 |                  0.02 |                      21 |
|   2 | (0, 7]      | (-0.9, -0.85]          |                     0 |                  0.07 |                   0.03 |                     0.02 |                  0.02 |                      20 |
|   3 | (0, 7]      | (-0.85, -0.8]          |                 -0.01 |                  0.06 |                   0.02 |                     0.02 |                  0.02 |                      27 |
|   4 | (0, 7]      | (-0.8, -0.75]          |                     0 |                  0.08 |                   0.03 |                     0.02 |                  0.02 |                      27 |
|   5 | (0, 7]      | (-0.75, -0.7]          |                 -0.01 |                  0.08 |                   0.03 |                     0.02 |                  0.02 |                      33 |
|   6 | (0, 7]      | (-0.7, -0.65]          |                 -0.01 |                  0.09 |                   0.03 |                     0.02 |                  0.03 |                      40 |
|   7 | (0, 7]      | (-0.65, -0.6]          |                 -0.01 |                  0.09 |                   0.03 |                     0.02 |                  0.03 |                      37 |
|   8 | (0, 7]      | (-0.6, -0.55]          |                 -0.02 |                  0.09 |                   0.03 |                     0.02 |                  0.03 |                      73 |
|   9 | (0, 7]      | (-0.55, -0.5]          |                 -0.02 |                   0.1 |                   0.03 |                     0.03 |                  0.03 |                     158 |
|  10 | (0, 7]      | (-0.5, -0.45]          |                 -0.02 |                  0.11 |                   0.03 |                     0.03 |                  0.03 |                     184 |
|  11 | (0, 7]      | (-0.45, -0.4]          |                 -0.02 |                  0.12 |                   0.04 |                     0.03 |                  0.03 |                     201 |
|  12 | (0, 7]      | (-0.4, -0.35]          |                 -0.02 |                  0.13 |                   0.04 |                     0.04 |                  0.04 |                     247 |
|  13 | (0, 7]      | (-0.35, -0.3]          |                 -0.02 |                  0.15 |                   0.05 |                     0.04 |                  0.04 |                     296 |
|  14 | (0, 7]      | (-0.3, -0.25]          |                 -0.03 |                  0.17 |                   0.05 |                     0.05 |                  0.05 |                     329 |
|  15 | (0, 7]      | (-0.25, -0.2]          |                 -0.03 |                   0.2 |                   0.06 |                     0.05 |                  0.05 |                     352 |
|  16 | (0, 7]      | (-0.2, -0.15]          |                 -0.04 |                  0.26 |                   0.08 |                     0.07 |                  0.07 |                     383 |
|  17 | (0, 7]      | (-0.15, -0.1]          |                 -0.06 |                  0.37 |                   0.11 |                     0.09 |                  0.09 |                     417 |
|  18 | (0, 7]      | (-0.1, -0.05]          |                 -0.12 |                  0.69 |                   0.18 |                     0.15 |                  0.16 |                     461 |
|  19 | (0, 7]      | (-0.05, 0.0]           |                    -1 |                  7.62 |                   0.64 |                     0.37 |                  1.03 |                     505 |
|  20 | (0, 7]      | (0.0, 0.05]            |                    -1 |                    68 |                   2.34 |                    -0.89 |                  8.65 |                     269 |
|  21 | (0, 7]      | (0.05, 0.1]            |                    -1 |                    -1 |                     -1 |                       -1 |                     0 |                       2 |
|  22 | (7, 14]     | (-1.0, -0.95]          |                  0.01 |                  0.09 |                   0.05 |                     0.04 |                  0.03 |                      12 |
|  23 | (7, 14]     | (-0.95, -0.9]          |                  0.01 |                   0.1 |                   0.05 |                     0.05 |                  0.02 |                      22 |
|  24 | (7, 14]     | (-0.9, -0.85]          |                  0.01 |                  0.09 |                   0.05 |                     0.05 |                  0.02 |                      16 |
|  25 | (7, 14]     | (-0.85, -0.8]          |                  0.01 |                  0.11 |                   0.06 |                     0.06 |                  0.03 |                      20 |
|  26 | (7, 14]     | (-0.8, -0.75]          |                  0.01 |                  0.11 |                   0.06 |                     0.06 |                  0.03 |                      25 |
|  27 | (7, 14]     | (-0.75, -0.7]          |                  0.01 |                  0.12 |                   0.06 |                     0.06 |                  0.03 |                      21 |
|  28 | (7, 14]     | (-0.7, -0.65]          |                  0.02 |                  0.12 |                   0.07 |                     0.06 |                  0.02 |                      33 |
|  29 | (7, 14]     | (-0.65, -0.6]          |                  0.02 |                  0.13 |                   0.07 |                     0.07 |                  0.03 |                      27 |
|  30 | (7, 14]     | (-0.6, -0.55]          |                  0.02 |                  0.14 |                   0.07 |                     0.07 |                  0.03 |                      45 |
|  31 | (7, 14]     | (-0.55, -0.5]          |                  0.02 |                  0.15 |                   0.07 |                     0.07 |                  0.03 |                     114 |
|  32 | (7, 14]     | (-0.5, -0.45]          |                  0.02 |                  0.16 |                   0.08 |                     0.08 |                  0.03 |                     153 |
|  33 | (7, 14]     | (-0.45, -0.4]          |                  0.02 |                  0.17 |                   0.09 |                     0.08 |                  0.04 |                     166 |
|  34 | (7, 14]     | (-0.4, -0.35]          |                  0.02 |                  0.19 |                   0.09 |                     0.09 |                  0.04 |                     197 |
|  35 | (7, 14]     | (-0.35, -0.3]          |                  0.02 |                  0.21 |                   0.11 |                      0.1 |                  0.04 |                     235 |
|  36 | (7, 14]     | (-0.3, -0.25]          |                  0.03 |                  0.25 |                   0.13 |                     0.12 |                  0.05 |                     265 |
|  37 | (7, 14]     | (-0.25, -0.2]          |                  0.03 |                   0.3 |                   0.15 |                     0.14 |                  0.06 |                     280 |
|  38 | (7, 14]     | (-0.2, -0.15]          |                  0.04 |                  0.38 |                   0.18 |                     0.18 |                  0.08 |                     307 |
|  39 | (7, 14]     | (-0.15, -0.1]          |                  0.05 |                  0.54 |                   0.25 |                     0.24 |                  0.11 |                     332 |
|  40 | (7, 14]     | (-0.1, -0.05]          |                  0.07 |                  0.97 |                    0.4 |                     0.39 |                  0.18 |                     370 |
|  41 | (7, 14]     | (-0.05, 0.0]           |                 -0.46 |                   4.4 |                   1.02 |                     0.86 |                  0.68 |                     404 |
|  42 | (7, 14]     | (0.0, 0.05]            |                    -1 |                    32 |                   1.52 |                    -0.73 |                  4.45 |                     388 |
|  43 | (7, 14]     | (0.05, 0.1]            |                    -1 |                 -0.83 |                  -0.93 |                    -0.94 |                  0.06 |                      36 |

There are more customization options for Optopsy's strategy functions, consult the codebase/future documentation to see how it can be used to adjust the results, such as increasing/decreasing
the intervals and other data to be returned.

The library is currently under development, as such expect changes to the API in the future.

