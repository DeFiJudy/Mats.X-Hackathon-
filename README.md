# Mats.X-Hackathon-
A Binary Options Game Proposal For Drip and MatsFi Economy.


# Project Name:  
Mats.Ex : A Real-Time Simulated Binary Options Trading Experience


# Introduction: 
This proposal introduces a simple to use P2E binary options game that allows users to apply their knowledge of the crypto markets in exchange for drip rewards. This game is not only fun and engaging but also educational and newbie-friendly , providing a learning experience for players with various skill levels.

 
# KEY FEATURES:
1. A Binary Option Trading Game:
A simulated trading interface where players make predictions (binary choices) about price movements. (Example: If the price of bitcoin increases within the next one minute the user makes a 92-100% profit and if it does not the user loses all his capital).

2. Real time price action:
The trading bot is able to dictate the prices of the assets in real time using API data from Coindesk.

3. Financial Interface: Key calls.
   
   Debit: Deducts Mats points from the player's balance when they encounter a loss.

   Credit: Adds Mats rewards when a player closes a trade in profit.

   Transfer: Enables users to move mats and fund their trading wallets and bets.

   Check Balance: Allows users to check their trading balance in real time.


# TARGET GROUP:
Mats.X is aimed to serve as an educational and entertainment tool for a wide range of users from new crypto /discord users to seasoned crypto Traders and enthusiasts.


# UNIQUE VALUE PROPOSITION:
While there are many demo trading platforms out there , there aren't many of them that allow users to explore trading concepts  while still earning rewards for their demo trading activities.
Secondly, Mats.X serves as an educational and entertainment tool for community members by enabling them to acquire risk-management, quick decision making, psychological and trading skills, striking a balance between entertainment and education through it's simple to use interface and intrinsic gambling feature.


# MILSTONES:
1. Introduction of more assets:
   A wider asset range will be introduced subsequntly to increase the betting options for users 
3. User-Friendly Dashboard:
   A dashboard with a clean interface to show current balance, trade history, and overall performance and net profit like the trading analytics feature 
   in binance. Displays win/loss ratios and provides graphical analytics for trading trends
4. Daily Refill feature:
   Introduction of a daily salary for users to claim in order to encourage daily interaction and a refill for users that may be low on trading points.
6. Invite feature:
   This feature will be added to reward members who invite other their friends to engage the trading game.
8. Introduction of higher timeframes
   Higher time frames ranging from  one day to a few months will be introduced for longterm traders , the higher the time frame the more the reward due to higher risk factors.

# REQUIREMENT FROM MEZO/DRIP:
  Provision of technical resources to help Pizza Labs create a better version of this game with the intended additional features above.

  # TEAM INFORMATION
  Disord ID : Judy#7245
  Email : kheity16@gmail.com
  Twitter: defi_judy
  






# project code.


```python
from discord.ext import commands, tasks
import discord
from discord import app_commands
import aiohttp
import datetime
import asyncio
import logging

Helper functions and decorators
def is_admin():
    def predicate(interaction: discord.Interaction) -> bool:
        return interaction.user.guild_permissions.administrator
    return app_commands.check(predicate)

class Economy(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot
        self.points_manager = bot.points_manager
        self.bets = {}  # Dictionary to store current bets (user_id -> bet info)
        self.current_round_end = None  # Store end time of the current round
        self.bet_open = False  # Flag to manage if betting is open
        self.start_price = None  # Store the starting price of Bitcoin for the current round
        self.user_bet_prices = {}  # Dictionary to store BTC price at the time of each user's bet
        self.check_betting_status.start()  # Start the task to check betting status
        self.price_update.start()  # Start the task to send price updates
        self.announcement_channel_id = 1302997967179087954  # Channel ID for automatic announcements (set this to your desired channel ID)
        logging.basicConfig(level=logging.INFO)

    @app_commands.guild_only()
    @app_commands.command(name="open_betting", description="[Admin] Open betting for the next round")
    @is_admin()
    async def open_betting(self, interaction: discord.Interaction, duration: int):
        """
        Opens betting for a new round with a specified duration in minutes.
        """
        if self.bet_open:
            await interaction.response.send_message("Betting is already open!", ephemeral=True)
            return

        if duration > 5:
            await interaction.response.send_message("The maximum betting duration is 5 minutes!", ephemeral=True)
            return

        self.bet_open = True
        self.current_round_end = datetime.datetime.now() + datetime.timedelta(minutes=duration)

        # Fetch the current Bitcoin price to use as the starting price
        async with aiohttp.ClientSession() as session:
            async with session.get("https://api.coindesk.com/v1/bpi/currentprice/BTC.json") as response:
                if response.status == 200:
                    data = await response.json()
                    self.start_price = float(data['bpi']['USD']['rate'].replace(',', ''))
                    await interaction.response.send_message(
                        f"🚨 Betting is NOW OPEN! You have {duration} minutes to place your bets! Current BTC price: ${self.start_price:.2f}",
                        ephemeral=False
                    )
                    logging.info(f"Betting opened with BTC price: ${self.start_price}")
                else:
                    await interaction.response.send_message(
                        "Failed to fetch the current Bitcoin price. Please try again later.",
                        ephemeral=True
                    )
                    self.bet_open = False

    @app_commands.guild_only()
    @app_commands.command(name="bet", description="Place a bet on whether Bitcoin will go up or down")
    @app_commands.describe(
        direction="Bet 'up' if you think Bitcoin price will go up, 'down' if it will go down.",
        amount="Amount of Points to bet (maximum 1000)",
        leverage="Leverage multiplier for your bet (e.g., 1x, 2x, up to 200x)"
    )
    async def place_bet(self, interaction: discord.Interaction, direction: str, amount: int, leverage: int):
        """
        Allows users to place a bet on the direction of Bitcoin price with an optional leverage.
        """
        if not self.bet_open:
            await interaction.response.send_message("Betting is currently closed! Stay tuned for the next round!", ephemeral=True)
            return

        if direction.lower() not in ["up", "down"]:
            await interaction.response.send_message("❌ Invalid direction! Please bet 'up' or 'down'.", ephemeral=True)
            return

        if amount <= 0 or amount > 1000:
            await interaction.response.send_message("❌ Amount must be positive and no more than 1000 Points!", ephemeral=True)
            return

        if leverage <= 0 or leverage > 200:
            await interaction.response.send_message("❌ Leverage must be between 1x and 200x!", ephemeral=True)
            return

        try:
            # Check if user has enough balance
            balance = await self.points_manager.get_balance(interaction.user.id)
            if balance < amount:
                await interaction.response.send_message(
                    f"❌ You don't have enough Points! Your balance: {balance:,} Points",
                    ephemeral=True
                )
                return

            # Fetch the current Bitcoin price to use as the bet price
            async with aiohttp.ClientSession() as session:
                async with session.get("https://api.coindesk.com/v1/bpi/currentprice/BTC.json") as response:
                    if response.status == 200:
                        data = await response.json()
                        bet_price = float(data['bpi']['USD']['rate'].replace(',', ''))
                    else:
                        await interaction.response.send_message(
                            "Failed to fetch the current Bitcoin price. Please try again later.",
                            ephemeral=True
                        )
                        return

            # Store the bet
            if interaction.user.id not in self.bets:
                self.bets[interaction.user.id] = []

            self.bets[interaction.user.id].append({
                "direction": direction.lower(),
                "amount": amount,
                "bet_price": bet_price,
                "leverage": leverage
            })

            # Deduct the bet amount from user balance
            await self.points_manager.remove_points(interaction.user.id, amount)

            await interaction.response.send_message(
                f"✅ Bet placed! You have bet {amount:,} Points on '{direction.upper()}' with {leverage}x leverage on BTC at a price of ${bet_price:.2f}. Good luck!",
                ephemeral=False
            )
            logging.info(f"User {interaction.user.id} placed a bet of {amount} Points on '{direction.lower()}' at BTC price ${bet_price} with {leverage}x leverage")
        except Exception as e:
            logging.error(f"Error placing bet for user {interaction.user.id}: {str(e)}")
            await interaction.response.send_message(f"❌ Error placing bet: {str(e)}", ephemeral=True)

    @app_commands.guild_only()
    @app_commands.command(name="close_betting", description="[Admin] Close betting and determine the outcome")
    @is_admin()
    async def close_betting(self, interaction: discord.Interaction):
        """
        Closes betting, determines the outcome, and rewards winners.
        """
        if not self.bet_open:
            await interaction.response.send_message("Betting is already closed!", ephemeral=True)
            return

        await self._close_betting_process(interaction)

    async def _close_betting_process(self, interaction: discord.Interaction = None):
        """
        Internal method to close betting and determine outcome, either manually or automatically.
        """
        # Close betting
        self.bet_open = False

        # Fetch the current Bitcoin price to determine the outcome
        async with aiohttp.ClientSession() as session:
            async with session.get("https://api.coindesk.com/v1/bpi/currentprice/BTC.json") as response:
                if response.status == 200:
                    data = await response.json()
                    end_price = float(data['bpi']['USD']['rate'].replace(',', ''))

                    # Determine winners and calculate winnings based on percentage change
                    winners = []
                    user_feedback = []

                    for user_id, bets in self.bets.items():
                        total_net_winnings = 0
                        user_wins = []

                        for bet in bets:
                            bet_price = bet["bet_price"]
                            price_difference = ((end_price - bet_price) / bet_price) * 100
                            outcome = "up" if end_price > bet_price else "down"

                            if bet["direction"] == outcome:
                                # Scale payout to leverage times the percentage difference in BTC price
                                winnings = bet["amount"] * (1 + abs(price_difference) * bet["leverage"] / 100)
                                winnings = max(int(winnings), 1)  # Minimum payout is 1 point
                                net_winnings = winnings - bet["amount"]
                                total_net_winnings += net_winnings
                                user_wins.append((winnings, net_winnings, price_difference, bet["leverage"]))
                                if winnings == 1:
                                    await self.points_manager.add_points(user_id, bet["amount"])
                                    user_feedback.append(f"<@{user_id}>, your bet resulted in the minimum payout of 1 point due to a small price movement.")
                            else:
                                points_lost = min(bet["amount"], bet["amount"] * (abs(price_difference) * bet["leverage"] / 100))
                                 # Refund the initial bet amount
                                await self.points_manager.add_points(user_id, bet["amount"])
                                # Deduct the calculated points lost
                                await self.points_manager.remove_points(user_id, int(points_lost))
                                user_feedback.append(f"<@{user_id}> lost {points_lost:.0f} mats !")

                        if total_net_winnings > 0:
                            winners.append((user_id, user_wins, total_net_winnings))

                    # Payout winners
                    for user_id, user_wins, total_net_winnings in winners:
                        total_winnings = sum(win[0] for win in user_wins)
                        await self.points_manager.add_points(user_id, total_winnings)

                    # Clear bets for the next round
                    self.bets.clear()

                    # Prepare a detailed winner message
                    if winners:
                        winners_details = "\n".join(
                            [
                                f"<@{user_id}> won {total_net_winnings:,} mats !"
                                for user_id, user_wins, total_net_winnings in winners
                            ]
                        )
                        message = (
                            f"🔒 Betting is now CLOSED! The Bitcoin price went to ${end_price:.2f}.\n"
                            f"🎉 We have {len(winners)} winner(s)! Congratulations! 💸\n\n"
                            f"🏆 Winners:\n{winners_details}"
                        )
                    else:
                        message = (
                            f"🔒 Betting is now CLOSED! The Bitcoin price went to ${end_price:.2f}.\n"
                            f"No winners this round. Better luck next time!"
                        )

                    # Include user feedback
                    if user_feedback:
                        feedback_message = "\n\n".join(user_feedback)
                        message += f"\n\n📢 Feedback:\n{feedback_message}"

                    logging.info(f"Betting closed. Outcome determined with BTC price at ${end_price}. {len(winners)} winners rewarded.")
                    if interaction:
                        await interaction.response.send_message(message, ephemeral=False)
                    else:
                        channel = self.bot.get_channel(self.announcement_channel_id)  # Replace with your channel ID
                        if channel:
                            await channel.send(message)
                else:
                    if interaction:
                        await interaction.response.send_message(
                            "Failed to fetch the current Bitcoin price. Please try again later.",
                            ephemeral=True
                        )
                    logging.error("Failed to fetch the current Bitcoin price during betting close process.")

    @tasks.loop(seconds=30)
    async def check_betting_status(self):
        """
        Periodically checks if the betting round should be closed automatically.
        """
        if self.bet_open and self.current_round_end and datetime.datetime.now() >= self.current_round_end:
            logging.info("Automatically closing betting round due to time expiration.")
            await self._close_betting_process()

    @tasks.loop(seconds=30)
    async def price_update(self):
        """
        Periodically sends the current Bitcoin price to the betting channel, along with the time left in the betting round.
        """
        if self.bet_open:
            # Calculate the remaining time for the betting round
            if self.current_round_end:
                time_left = self.current_round_end - datetime.datetime.now()
                minutes, seconds = divmod(int(time_left.total_seconds()), 60)
                time_left_str = f"{minutes} minute(s) and {seconds} second(s)" if time_left.total_seconds() > 0 else "0 seconds"

            # Fetch the current Bitcoin price
            async with aiohttp.ClientSession() as session:
                async with session.get("https://api.coindesk.com/v1/bpi/currentprice/BTC.json") as response:
                    if response.status == 200:
                        data = await response.json()
                        current_price = float(data['bpi']['USD']['rate'].replace(',', ''))
                        channel = self.bot.get_channel(self.announcement_channel_id)  # Replace with your channel ID
                        if channel:
                            await channel.send(
                                f"📈 Current BTC Price Update: ${current_price:.2f}\n"
                                f"⏳ Time left in betting round: {time_left_str}"
                            )
                            logging.info(f"Price update sent: ${current_price}, time left: {time_left_str}")
                    else:
                        logging.error("Failed to fetch the current Bitcoin price for update.")


    @app_commands.guild_only()
    @app_commands.command(name="my_bet", description="Check your current bet for the ongoing round")
    async def my_bet(self, interaction: discord.Interaction):
        """
        Allows users to check their current bet for the ongoing round.
        """
        bets = self.bets.get(interaction.user.id)
        if not bets:
            await interaction.response.send_message("You haven't placed a bet this round! Place your bets while you still can!", ephemeral=True)
        else:
            bet_details = "\n".join(
                [
                    f"💰 Bet {i+1}: {bet['amount']:,} Points on '{bet['direction'].upper()}' with {bet['leverage']}x leverage at BTC price ${bet['bet_price']:.2f}"
                    for i, bet in enumerate(bets)
                ]
            )
            await interaction.response.send_message(
                f"Here are your current bets:\n{bet_details}\nGood luck! 🍀",
                ephemeral=True
            )

    @open_betting.error
    @close_betting.error
    async def admin_error(self, interaction: discord.Interaction, error: app_commands.AppCommandError):
        if isinstance(error, app_commands.CheckFailure):
            await interaction.response.send_message(
                "❌ You don't have permission to use this command!",
                ephemeral=True
            )

async def setup(bot: commands.Bot) -> None:
    await bot.add_cog(Economy(bot))
