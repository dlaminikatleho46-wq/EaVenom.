# EaVenom.
//+------------------------------------------------------------------+
//| EaVenom.mq5                                                      |
//| Founder: Katleho Dlamini                                        |
//| EMA 50 / EMA 200 multi-pair paper trading scanner                |
//+------------------------------------------------------------------+
#property strict
#property version "1.00"
#property description "EaVenom - LIMO EDGE inspired EMA scanner"

input ENUM_TIMEFRAMES ScanTimeframe = PERIOD_M15;
input int FastEMA = 50;
input int SlowEMA = 200;
input int MinProfitPoints = 10;
input int MaxLossPoints = 300;
input int ScanIntervalSeconds = 2;

struct PairData
{
   string symbol;
   int fastHandle;
   int slowHandle;
   int position;       // 0=none, 1=buy, -1=sell
   double entryPrice;
   datetime lastBar;
   string status;
};

PairData pairs[];
int pairCount = 0;

//+------------------------------------------------------------------+
int OnInit()
{
   int total = SymbolsTotal(true);

   ArrayResize(pairs, total);

   for(int i = 0; i < total; i++)
   {
      string sym = SymbolName(i, true);

      if(!SymbolSelect(sym, true))
         continue;

      pairs[pairCount].symbol = sym;
      pairs[pairCount].fastHandle =
         iMA(sym, ScanTimeframe, FastEMA, 0, MODE_EMA, PRICE_CLOSE);

      pairs[pairCount].slowHandle =
         iMA(sym, ScanTimeframe, SlowEMA, 0, MODE_EMA, PRICE_CLOSE);

      pairs[pairCount].position = 0;
      pairs[pairCount].entryPrice = 0;
      pairs[pairCount].lastBar = 0;
      pairs[pairCount].status = "Scanning";

      if(pairs[pairCount].fastHandle == INVALID_HANDLE ||
         pairs[pairCount].slowHandle == INVALID_HANDLE)
      {
         Print("Indicator error: ", sym);
         continue;
      }

      pairCount++;
   }

   ArrayResize(pairs, pairCount);

   if(pairCount == 0)
   {
      Print("No valid Market Watch symbols found.");
      return INIT_FAILED;
   }

   EventSetTimer(MathMax(1, ScanIntervalSeconds));

   Print("EaVenom started. Pairs scanned: ", pairCount);
   return INIT_SUCCEEDED;
}

//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   EventKillTimer();

   for(int i = 0; i < pairCount; i++)
   {
      if(pairs[i].fastHandle != INVALID_HANDLE)
         IndicatorRelease(pairs[i].fastHandle);

      if(pairs[i].slowHandle != INVALID_HANDLE)
         IndicatorRelease(pairs[i].slowHandle);
   }

   Comment("");
}

//+------------------------------------------------------------------+
void OnTick()
{
   // Scanning is performed by OnTimer.
}

//+------------------------------------------------------------------+
void OnTimer()
{
   string dashboard = "";
   dashboard += "================================\n";
   dashboard += "          EA VENOM\n";
   dashboard += "    Founder: Katleho Dlamini\n";
   dashboard += "       LIMO EDGE STYLE\n";
   dashboard += "================================\n";
   dashboard += "Mode: PAPER TRADING ONLY\n";
   dashboard += "Strategy: EMA 50 / EMA 200\n";
   dashboard += "Timeframe: M15\n\n";

   for(int i = 0; i < pairCount; i++)
   {
      string sym = pairs[i].symbol;

      MqlRates rates[];
      double fast[];
      double slow[];

      ArraySetAsSeries(rates, true);
      ArraySetAsSeries(fast, true);
      ArraySetAsSeries(slow, true);

      if(CopyRates(sym, ScanTimeframe, 0, 3, rates) < 3)
         continue;

      if(CopyBuffer(pairs[i].fastHandle, 0, 0, 3, fast) < 3)
         continue;

      if(CopyBuffer(pairs[i].slowHandle, 0, 0, 3, slow) < 3)
         continue;

      bool newBar = false;

      if(rates[0].time != pairs[i].lastBar)
      {
         pairs[i].lastBar = rates[0].time;
         newBar = true;
      }

      double bid = 0, ask = 0, point = 0;

      if(!SymbolInfoDouble(sym, SYMBOL_BID, bid) ||
         !SymbolInfoDouble(sym, SYMBOL_ASK, ask))
         continue;

      point = SymbolInfoDouble(sym, SYMBOL_POINT);

      if(point <= 0)
         continue;

      // Manage existing virtual position.
      if(pairs[i].position == 1)
      {
         double profitPoints =
            (bid - pairs[i].entryPrice) / point;

         if(profitPoints >= MinProfitPoints)
         {
            pairs[i].position = 0;
            pairs[i].status = "BUY CLOSED: PROFIT";
         }
         else if(profitPoints <= -MaxLossPoints)
         {
            pairs[i].position = 0;
            pairs[i].status = "BUY CLOSED: LOSS LIMIT";
         }
      }
      else if(pairs[i].position == -1)
      {
         double profitPoints =
            (pairs[i].entryPrice - ask) / point;

         if(profitPoints >= MinProfitPoints)
         {
            pairs[i].position = 0;
            pairs[i].status = "SELL CLOSED: PROFIT";
         }
         else if(profitPoints <= -MaxLossPoints)
         {
            pairs[i].position = 0;
            pairs[i].status = "SELL CLOSED: LOSS LIMIT";
         }
      }

      // Only enter on a fresh candle.
      if(newBar && pairs[i].position == 0)
      {
         bool bullish = fast[1] > slow[1];
         bool bearish = fast[1] < slow[1];

         // Pullback followed by bullish recovery.
         bool buySignal =
            bullish &&
            rates[2].close <= fast[2] &&
            rates[1].close > fast[1];

         // Pullback followed by bearish recovery.
         bool sellSignal =
            bearish &&
            rates[2].close >= fast[2] &&
            rates[1].close < fast[1];

         if(buySignal)
         {
            pairs[i].position = 1;
            pairs[i].entryPrice = ask;
            pairs[i].status = "VIRTUAL BUY OPEN";
         }
         else if(sellSignal)
         {
            pairs[i].position = -1;
            pairs[i].entryPrice = bid;
            pairs[i].status = "VIRTUAL SELL OPEN";
         }
         else
         {
            pairs[i].status = "WAITING FOR SIGNAL";
         }
      }

      string trade = "NONE";

      if(pairs[i].position == 1)
         trade = "BUY";

      if(pairs[i].position == -1)
         trade = "SELL";

      dashboard += sym + " | " + trade +
                   " | " + pairs[i].status + "\n";
   }

   Comment(dashboard);
}
//+------------------------------------------------------------------+
