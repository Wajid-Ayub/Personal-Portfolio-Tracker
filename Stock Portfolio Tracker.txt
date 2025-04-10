import yfinance as yf
import pandas as pd
from datetime import datetime
import json
import os
import tkinter as tk
from tkinter import ttk, messagebox
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

class StockPortfolioGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Stock Portfolio Tracker")
        self.root.geometry("1100x750")
        self.root.configure(bg="#2c3e50")  # Dark blue-gray background
        
        self.portfolio = StockPortfolio()
        self.create_gui()

    def create_gui(self):
        # Style configuration
        style = ttk.Style()
        style.theme_use('clam')
        
        # Custom styles
        style.configure("TButton", font=("Helvetica", 11, "bold"), padding=8, background="#3498db", 
                       foreground="white", borderwidth=0)
        style.map("TButton", background=[('active', '#2980b9')])
        
        style.configure("TLabel", font=("Helvetica", 12), background="#2c3e50", foreground="white")
        style.configure("Header.TLabel", font=("Helvetica", 18, "bold"), foreground="#ecf0f1")
        
        style.configure("Treeview", font=("Helvetica", 11), rowheight=25, background="#34495e",
                       foreground="white", fieldbackground="#34495e")
        style.configure("Treeview.Heading", font=("Helvetica", 12, "bold"), background="#2980b9",
                       foreground="white")

        # Header
        header = ttk.Label(self.root, text="Stock Portfolio Tracker", style="Header.TLabel")
        header.pack(pady=20)

        # Main Frame
        main_frame = ttk.Frame(self.root, style="Card.TFrame")
        main_frame.pack(pady=10, padx=20, fill="both", expand=True)

        # Input Frame
        input_frame = ttk.Frame(main_frame, style="Card.TFrame")
        input_frame.pack(pady=10, padx=10, fill="x")

        # Add Stock Section with modern entries
        ttk.Label(input_frame, text="Symbol:").grid(row=0, column=0, padx=10, pady=5)
        self.symbol_entry = ttk.Entry(input_frame, font=("Helvetica", 11), width=15)
        self.symbol_entry.grid(row=0, column=1, padx=10, pady=5)

        ttk.Label(input_frame, text="Shares:").grid(row=0, column=2, padx=10, pady=5)
        self.shares_entry = ttk.Entry(input_frame, font=("Helvetica", 11), width=15)
        self.shares_entry.grid(row=0, column=3, padx=10, pady=5)

        ttk.Label(input_frame, text="Buy Price:").grid(row=0, column=4, padx=10, pady=5)
        self.price_entry = ttk.Entry(input_frame, font=("Helvetica", 11), width=15)
        self.price_entry.grid(row=0, column=5, padx=10, pady=5)

        ttk.Button(input_frame, text="Add Stock", command=self.add_stock).grid(row=0, column=6, padx=10, pady=5)
        ttk.Button(input_frame, text="Remove Stock", command=self.remove_stock).grid(row=0, column=7, padx=10, pady=5)

        # Portfolio Display
        self.tree = ttk.Treeview(main_frame, columns=("Symbol", "Shares", "Buy Price", "Current", 
                                                     "Gain/Loss", "Value", "P/E", "Div Yield"),
                                show="headings", height=15)
        self.tree.pack(pady=10, padx=10, fill="both", expand=True)

        # Column headings and widths
        columns = {
            "Symbol": 100, "Shares": 80, "Buy Price": 100, "Current": 100,
            "Gain/Loss": 100, "Value": 100, "P/E": 80, "Div Yield": 100
        }
        for col, width in columns.items():
            self.tree.heading(col, text=col)
            self.tree.column(col, width=width, anchor="center")

        # Summary Frame
        summary_frame = ttk.Frame(main_frame, style="Card.TFrame")
        summary_frame.pack(pady=10, fill="x")

        self.total_value_label = ttk.Label(summary_frame, text="Total Value: $0.00", 
                                         font=("Helvetica", 12, "bold"), foreground="#2ecc71")
        self.total_value_label.pack(side="left", padx=15)
        
        self.roi_label = ttk.Label(summary_frame, text="ROI: 0.00%", 
                                 font=("Helvetica", 12, "bold"), foreground="#2ecc71")
        self.roi_label.pack(side="left", padx=15)

        # Buttons Frame
        buttons_frame = ttk.Frame(main_frame)
        buttons_frame.pack(pady=15)

        ttk.Button(buttons_frame, text="Refresh", command=self.refresh_portfolio).pack(side="left", padx=10)
        ttk.Button(buttons_frame, text="Show Graph", command=self.show_graph).pack(side="left", padx=10)
        ttk.Button(buttons_frame, text="Exit", command=self.root.quit, 
                  style="Danger.TButton").pack(side="left", padx=10)

        # Custom style for exit button
        style.configure("Danger.TButton", background="#e74c3c")
        style.map("Danger.TButton", background=[('active', '#c0392b')])

        self.refresh_portfolio()

    def add_stock(self):
        try:
            symbol = self.symbol_entry.get().upper()
            shares = float(self.shares_entry.get())
            buy_price = float(self.price_entry.get())
            
            self.portfolio.add_stock(symbol, shares, buy_price)
            self.refresh_portfolio()
            
            self.symbol_entry.delete(0, tk.END)
            self.shares_entry.delete(0, tk.END)
            self.price_entry.delete(0, tk.END)
        except ValueError:
            messagebox.showerror("Error", "Please enter valid numerical values!", parent=self.root)

    def remove_stock(self):
        symbol = self.symbol_entry.get().upper()
        if symbol:
            self.portfolio.remove_stock(symbol)
            self.refresh_portfolio()
            self.symbol_entry.delete(0, tk.END)

    def refresh_portfolio(self):
        for item in self.tree.get_children():
            self.tree.delete(item)

        total_value = 0
        total_gain_loss = 0
        total_invested = 0

        for symbol, data in self.portfolio.portfolio.items():
            stock_data = self.portfolio.get_stock_data(symbol)
            if stock_data is None or stock_data['price'] is None:
                continue

            shares = data['shares']
            buy_price = data['buy_price']
            current_price = stock_data['price']
            pe_ratio = stock_data['pe_ratio']
            div_yield = stock_data['dividend_yield']

            initial_value = shares * buy_price
            current_value = shares * current_price
            gain_loss = current_value - initial_value

            total_value += current_value
            total_gain_loss += gain_loss
            total_invested += initial_value

            pe_display = f"{pe_ratio:.2f}" if pe_ratio else "N/A"
            div_display = f"{div_yield:.2f}%" if div_yield else "0.00%"

            gain_loss_color = "#2ecc71" if gain_loss >= 0 else "#e74c3c"
            self.tree.insert("", "end", values=(
                symbol, shares, f"${buy_price:.2f}", f"${current_price:.2f}",
                f"${gain_loss:.2f}", f"${current_value:.2f}", pe_display, div_display
            ))

        roi = (total_gain_loss / total_invested * 100) if total_invested > 0 else 0
        self.total_value_label.config(text=f"Total Value: ${total_value:.2f}")
        self.roi_label.config(text=f"ROI: {roi:.2f}%")

    def show_graph(self):
        if not self.portfolio.portfolio:
            messagebox.showinfo("Info", "Portfolio is empty!", parent=self.root)
            return

        graph_window = tk.Toplevel(self.root)
        graph_window.title("Portfolio Performance Graph")
        graph_window.geometry("900x650")
        graph_window.configure(bg="#2c3e50")

        fig, ax = plt.subplots(figsize=(12, 6), facecolor="#34495e")
        ax.set_facecolor("#34495e")
        
        symbols = list(self.portfolio.portfolio.keys())
        values = []
        for symbol in symbols:
            stock_data = self.portfolio.get_stock_data(symbol)
            if stock_data and stock_data['price']:
                values.append(self.portfolio.portfolio[symbol]['shares'] * stock_data['price'])
            else:
                values.append(0)

        bars = ax.bar(symbols, values, color='#3498db', edgecolor='white')
        ax.set_title("Portfolio Holdings Value", color="white", fontsize=16, pad=15)
        ax.set_xlabel("Stock Symbols", color="white", fontsize=12)
        ax.set_ylabel("Value ($)", color="white", fontsize=12)
        
        ax.tick_params(axis='x', colors='white', rotation=45)
        ax.tick_params(axis='y', colors='white')
        
        # Add value labels on top of bars
        for bar in bars:
            height = bar.get_height()
            ax.text(bar.get_x() + bar.get_width()/2., height,
                   f'${height:.2f}', ha='center', va='bottom', color='white')

        canvas = FigureCanvasTkAgg(fig, master=graph_window)
        canvas.draw()
        canvas.get_tk_widget().pack(fill="both", expand=True, padx=10, pady=10)

class StockPortfolio:
    def __init__(self, filename="portfolio.json"):
        self.filename = filename
        self.portfolio = self.load_portfolio()

    def load_portfolio(self):
        if os.path.exists(self.filename):
            try:
                with open(self.filename, 'r') as f:
                    return json.load(f)
            except:
                return {}
        return {}

    def save_portfolio(self):
        with open(self.filename, 'w') as f:
            json.dump(self.portfolio, f, indent=4)

    def add_stock(self, symbol, shares, buy_price):
        symbol = symbol.upper()
        if shares <= 0:
            messagebox.showerror("Error", "Number of shares must be positive!")
            return
        
        if not self.verify_symbol(symbol):
            messagebox.showerror("Error", f"Invalid stock symbol: {symbol}")
            return

        self.portfolio[symbol] = {
            'shares': shares,
            'buy_price': buy_price,
            'added_date': datetime.now().strftime("%Y-%m-%d")
        }
        self.save_portfolio()
        messagebox.showinfo("Success", f"Added {shares} shares of {symbol} at ${buy_price:.2f}")

    def remove_stock(self, symbol):
        symbol = symbol.upper()
        if symbol in self.portfolio:
            del self.portfolio[symbol]
            self.save_portfolio()
            messagebox.showinfo("Success", f"Removed {symbol} from portfolio")
        else:
            messagebox.showerror("Error", f"{symbol} not found in portfolio")

    def verify_symbol(self, symbol):
        try:
            stock = yf.Ticker(symbol)
            return bool(stock.info['regularMarketPrice'])
        except:
            return False

    def get_stock_data(self, symbol):
        try:
            stock = yf.Ticker(symbol)
            info = stock.info
            return {
                'price': info.get('regularMarketPrice'),
                'pe_ratio': info.get('trailingPE'),
                'dividend_yield': info.get('dividendYield', 0) * 100 if info.get('dividendYield') else 0
            }
        except Exception as e:
            print(f"Error fetching data for {symbol}: {e}")
            return None

if __name__ == "__main__":
    root = tk.Tk()
    app = StockPortfolioGUI(root)
    root.mainloop()
    