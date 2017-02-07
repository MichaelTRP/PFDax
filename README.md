// Pathfinder Trading System based on ProRealTime 10.2
// Breakout system triggered by previous daily, weekly and monthly high/low crossings with smart position management
// Version 6 Beta 2
// Instrument: DAX mini 4H, 9-21 CET, 2 points spread, account size 10.000 Euro, from August 2010

// ProOrder code parameter
DEFPARAM CUMULATEORDERS = true  // cumulate orders if not turned off
DEFPARAM PRELOADBARS = 10000

// define intraday trading window
ONCE startTime = 80000
ONCE endTime = 200000

// define instrument signalline with help of multiple smoothed averages
ONCE periodFirstMA = 5
ONCE periodSecondMA = 10
ONCE periodThirdMA = 3

// define filter parameter
ONCE periodLongMA = 300
ONCE periodShortMA = 50

// define position and money management parameter
ONCE positionSize = 2

Capital = 10000
Risk = 5 // in %
equity = Capital + StrategyProfit
maxRisk = round(equity * Risk / 100)

ONCE stopLossLong = 5.5 // in %
ONCE stopLossShort = 3.5 // in %
ONCE takeProfitLong = 3.25 // in %
ONCE takeProfitShort = 2.75 // in %

maxPositionSizeLong = MAX(15, abs(round(maxRisk / (close * stopLossLong / 100) / PointValue) * pipsize))
maxPositionSizeShort = MAX(15, abs(round(maxRisk / (close * stopLossShort / 100) / PointValue) * pipsize))

ONCE trailingStartLong = 2 // in %
ONCE trailingStartShort = 0.75 // in %
ONCE trailingStepLong = 0.2 // in %
ONCE trailingStepShort = 0.4 // in %

ONCE maxCandlesLongWithProfit = 16  // take long profit latest after 16 candles
ONCE maxCandlesShortWithProfit = 15  // take short profit latest after 15 candles
ONCE maxCandlesLongWithoutProfit = 30  // limit long loss latest after 30 candles
ONCE maxCandlesShortWithoutProfit = 12  // limit short loss latest after 12 candles

// define saisonal position multiplier >0 - long / <0 - short / 0 no trade
ONCE January = 1
ONCE February = 1
ONCE March = 1
ONCE April = 1
ONCE May = 1
ONCE June = 1
ONCE July = 1
ONCE August = 1
ONCE September = 1
ONCE October = 1
ONCE November = 1
ONCE December = 1

// calculate daily high/low (include sunday values if available)
dailyHigh = DHigh(1)
dailyLow = DLow(1)

// calculate weekly high/low
If DayOfWeek < DayOfWeek[1] then
weeklyHigh = Highest[BarIndex - lastWeekBarIndex](dailyHigh)
//weeklyLow = Lowest[BarIndex - lastWeekBarIndex](dailyLow)
lastWeekBarIndex = BarIndex
ENDIF

// calculate monthly high/low
If Month <> Month[1] then
monthlyHigh = Highest[BarIndex - lastMonthBarIndex](dailyHigh)
monthlyLow = Lowest[BarIndex - lastMonthBarIndex](dailyLow)
lastMonthBarIndex = BarIndex
ENDIF

// calculate instrument signalline with multiple smoothed averages
firstMA = WilderAverage[periodFirstMA](close)
secondMA = TimeSeriesAverage[periodSecondMA](firstMA)
signalline = TimeSeriesAverage[periodThirdMA](secondMA)

// save position before trading window is open
If Time < startTime then
startPositionLong = COUNTOFLONGSHARES
startPositionShort = COUNTOFSHORTSHARES
EndIF

// trade only in defined trading window
IF Time >= startTime AND Time <= endTime THEN

// set saisonal pattern
IF CurrentMonth = 1 THEN
saisonalPatternMultiplier = January
ELSIF CurrentMonth = 2 THEN
saisonalPatternMultiplier = February
ELSIF CurrentMonth = 3 THEN
saisonalPatternMultiplier = March
ELSIF CurrentMonth = 4 THEN
saisonalPatternMultiplier = April
ELSIF CurrentMonth = 5 THEN
saisonalPatternMultiplier = May
ELSIF CurrentMonth = 6 THEN
saisonalPatternMultiplier = June
ELSIF CurrentMonth = 7 THEN
saisonalPatternMultiplier = July
ELSIF CurrentMonth = 8 THEN
saisonalPatternMultiplier = August
ELSIF CurrentMonth = 9 THEN
saisonalPatternMultiplier = September
ELSIF CurrentMonth = 10 THEN
saisonalPatternMultiplier = October
ELSIF CurrentMonth = 11 THEN
saisonalPatternMultiplier = November
ELSIF CurrentMonth = 12 THEN
saisonalPatternMultiplier = December
ENDIF

// define trading filters
// 1. use fast and slow averages as filter because not every breakout is profitable
f1 = close > Average[periodLongMA](close)
f2 = close < Average[periodLongMA](close)
f3 = close > Average[periodShortMA](close)

// 2. check if position already reduced in trading window as additonal filter criteria
alreadyReducedLongPosition = COUNTOFLONGSHARES  < startPositionLong
alreadyReducedShortPosition = COUNTOFSHORTSHARES < startPositionShort

// long position conditions
l1 = signalline CROSSES OVER monthlyHigh
l2 = signalline CROSSES OVER weeklyHigh
l3 = signalline CROSSES OVER dailyHigh
l4 = signalline CROSSES OVER monthlyLow

// short position conditions
s1 = signalline CROSSES UNDER monthlyHigh
s2 = signalline CROSSES UNDER dailyLow

// long entry with order cumulation
IF ( (l1 OR l4 OR l2 OR (l3 AND f2)) AND NOT alreadyReducedLongPosition) THEN

// check saisonal booster setup and max position size
IF saisonalPatternMultiplier > 0 THEN
IF (COUNTOFPOSITION + (positionSize * saisonalPatternMultiplier)) <= maxPositionSizeLong THEN
BUY positionSize * saisonalPatternMultiplier CONTRACT AT MARKET
ENDIF
ELSIF saisonalPatternMultiplier <> 0 THEN
IF (COUNTOFPOSITION + positionSize) <= maxPositionSizeLong THEN
BUY positionSize CONTRACT AT MARKET
ENDIF
ENDIF

stopLoss = stopLossLong
takeProfit = takeProfitLong

ENDIF

// short entry without order cumulation
IF NOT SHORTONMARKET AND ( (s1 AND f3) OR  (s2 AND f1) ) AND NOT alreadyReducedShortPosition THEN

// check saisonal booster setup and max position size
IF saisonalPatternMultiplier < 0 THEN
IF (COUNTOFPOSITION + (positionSize * ABS(saisonalPatternMultiplier))) <= maxPositionSizeShort THEN
SELLSHORT positionSize * ABS(saisonalPatternMultiplier) CONTRACT AT MARKET
ENDIF
ELSIF saisonalPatternMultiplier <> 0 THEN
IF (COUNTOFPOSITION + positionSize) <= maxPositionSizeLong THEN
SELLSHORT positionSize CONTRACT AT MARKET
ENDIF
ENDIF

stopLoss = stopLossShort
takeProfit = takeProfitShort

ENDIF

// stop and profit management
posProfit = (((close - positionprice) * pointvalue) * countofposition) / pipsize

numberCandles = (BarIndex - TradeIndex)

m1 = posProfit > 0 AND numberCandles >= maxCandlesLongWithProfit
m2 = posProfit > 0 AND numberCandles >= maxCandlesShortWithProfit
m3 = posProfit < 0 AND numberCandles >= maxCandlesLongWithoutProfit
m4 = posProfit < 0 AND numberCandles >= maxCandlesShortWithoutProfit

// take profit after max candles
IF LONGONMARKET AND (m1 OR m3) THEN
SELL AT MARKET
ENDIF
IF SHORTONMARKET AND (m2 OR m4) THEN
EXITSHORT AT MARKET
ENDIF

// trailing stop function
trailingStartLongInPoints = tradeprice(1) * trailingStartLong / 100
trailingStartShortInPoints = tradeprice(1) * trailingStartShort / 100
trailingStepLongInPoints = tradeprice(1) * trailingStepLong / 100
trailingStepShortInPoints = tradeprice(1) * trailingStepShort / 100

// reset the stoploss value
IF NOT ONMARKET THEN
newSL = 0
ENDIF

// manage long positions
IF LONGONMARKET THEN
// first move (breakeven)
IF newSL = 0 AND close - tradeprice(1) >= trailingStartLongInPoints * pipsize THEN
newSL = tradeprice(1) + trailingStepLongInPoints * pipsize
stopLoss = stopLossLong * 0.1
takeProfit = takeProfitLong * 2
ENDIF
// next moves
IF newSL > 0 AND close - newSL >= trailingStepLongInPoints * pipsize THEN
newSL = newSL + trailingStepLongInPoints * pipsize
ENDIF
ENDIF

// manage short positions
IF SHORTONMARKET THEN
// first move (breakeven)
IF newSL = 0 AND tradeprice(1) - close >= trailingStartShortInPoints * pipsize THEN
newSL = tradeprice(1) - trailingStepShortInPoints * pipsize
stopLoss = stopLossShort * 0.5
takeProfit = takeProfitShort * 1.3
ENDIF
// next moves
IF newSL > 0 AND newSL - close >= trailingStepShortInPoints * pipsize THEN
newSL = newSL - trailingStepShortInPoints * pipsize
ENDIF
ENDIF

// stop order to exit the positions
IF newSL > 0 THEN
IF LONGONMARKET THEN
SELL AT newSL STOP
ENDIF
IF SHORTONMARKET THEN
EXITSHORT AT newSL STOP
ENDIF
ENDIF

// superordinate stop and take profit
SET STOP %LOSS stopLoss
SET TARGET %PROFIT takeProfit

ENDIF
