# Introduction

`pymt5adapter` is a wrapper and drop-in replacement (wrapper) for the `MetaTrader5` python package by MetaQuotes. 
The API functions simply pass through values from the `MetaTrader5` functions, but adds the following functionality
in addition to a more pythonic interface:

 - Typing hinting has been added to all functions and return objects for linting and IDE integration. 
 Now intellisense will work no matter how nested objects are. ![alt text][intellisence_screen]
 - Docstrings have been added to each function 
 (see MQL [documentation](https://www.mql5.com/en/docs/integration/python_metatrader5)). 
 Docs can now be accessed on the fly in the IDE. For example: `Ctrl+Q` in pycharm. ![alt text][docs_screen]
 - All params can now be called by keyword. No more positional only args.
 - Testing included compliments of `pytest`
 - A new context manager has been added to provide a more pythonic interface to the setup and tear-down 
 of the terminal connection. The use of this context-manager can do the following: 
 
   - Ensures that `mt5.shutdown()` is always called, even if the user code throws an uncaught exception.
   - Can modify the entire behavior of the API with some simple settings:
      - Ensure the terminal has enabled auto-trading.
      - Prevents running on real account by default
      - Can automatically enable global debugging using your logger of choice
      - Can raise the custom `MT5Error `exception whenever `last_error()[0] != RES_S_OK` (off by default)


# Installation

```
pip install -U pymt5adapter
```
 
The `MetaTrader5` dependency sometimes has issues installing with the `pip` version that automatically gets 
packaged inside of the `virualenv` environment. If cannot install `MetaTrader5` then you need to update `pip` 
inside of the virtualenv. From the command line within the virual environment use:

```
(inside virtualenv):easy_install -U pip
```

# Import  
This should work with any existing `MetaTrader5` script by simply changing the `import` statement from:  
```python
import MetaTrader5 as mt5 
```
to:  
```python
import pymt5adapter as mt5 
``` 
     
# Context Manager

The `connected` function returns a context manager which performs all API setup and tear-down and ensures 
that `mt5.shutdown()` is always called. 

### Hello world

Using the context manager can be easy as...

```python
import pymt5adapter as mt5

with mt5.connected():
    print(mt5.version())

```

and can be customized to modify the entire API

```python
import pymt5adapter as mt5

mt5_connected = mt5.connected(
    path=r'C:\Users\user\Desktop\MT5\terminal64.exe',
    portable=True,
    server='MetaQuotes-Demo',
    login=1234567,
    password='password1',
    timeout=5000,
    ensure_trade_enabled=True, # default is False
    enable_real_trading=False, # default is False
    raise_on_errors=True,      # default is False
    debug_logging=True,        # default is False
    logger=print,              # default is print
)
with mt5_connected as conn:
    try:
        num_orders = mt5.history_orders_total("invalid", "arguments")
    except mt5.MT5Error as e:
        print("We modified the API to throw exceptions for all functions.")
        print(f"Error = {e}")
    # change error handling behavior at runtime
    conn.raise_on_errors = False
    try:
        num_orders = mt5.history_orders_total("invalid", "arguments")
    except mt5.MT5Error:
        pass
    else:
        print('We modified the API to silence Exceptions at runtime')

```

Output:

```
MT5 connection has been initialized.
[history_orders_total(invalid, arguments)][(-2, 'Invalid arguments')][None]
We modified the API to throw exceptions for all functions.
Error = (<ERROR_CODE.INVALID_PARAMS: -2>, "Invalid arguments('invalid', 'arguments'){}")
[history_orders_total(invalid, arguments)][(-2, 'Invalid arguments')][None]
We modified the API to silence Exceptions at runtime
[shutdown()][(1, 'Success')][True]
MT5 connection has been shutdown.

```

# Additional features not included in the `MetaTrader5` package

### Filter function callbacks

The following API functions can now except an optional callback for filtering using the named parameter, `function`:
* `symbols_get`
* `orders_get`
* `positions_get`
* `history_deals_get`
* `history_orders_get`

Example:

```python
visible_symbols = mt5.symbols_get(function=lambda s: s.visible)

def out_deal(deal: mt5.TradeDeal):
    return deal.entry == mt5.DEAL_ENTRY_OUT

out_deals = mt5.history_deals_get(function=out_deal)
```

### Calendar Events
Get economic calendar events from mql5.com using the `calendar_events` function. The function is memoized so
repeated calls pulls event data from cache instead of repeated calls to the server. 

Example:
```python
from pymt5adapter.calendar import calendar_events
from pymt5adapter.calendar import Currency
from pymt5adapter.calendar import Importance

symbol = mt5.symbols_get(function=forex_symbol)[0]
one_week = timedelta(weeks=1)
now = datetime.now()
default_one_week_ahead_all_events = calendar_events()
filtered_by_callback = calendar_events(function=lambda e: 'fed' in e['event_name'].lower())
filtered_by_flags = calendar_events(importance=Importance.MEDIUM | Importance.HIGH,
                                    currencies=Currency.USD | Currency.JPY)
filtered_by_strings = calendar_events(importance='medium high',
                                      currencies='usdjpy')
filtered_by_iterables = calendar_events(importance=('medium', ' high'),
                                        currencies=('usd', 'jpy'))
filtered_by_specific_times = calendar_events(time_from=now, time_to=now + one_week)
filtered_by_timedeltas = calendar_events(time_from=(-one_week), time_to=one_week)
filtered_by_timedelta_lookback = calendar_events(-one_week)
calendar_events_in_russian = calendar_events(language='ru')
```

[intellisence_screen]: https://github.com/nicholishen/pymt5adapter/raw/master/images/intellisense_screen.jpg "intellisence example"
[docs_screen]: https://github.com/nicholishen/pymt5adapter/raw/master/images/docs_screen.jpg "quick docs example"