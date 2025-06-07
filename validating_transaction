from datetime import datetime
from time import sleep
import requests
from xrpl.clients import JsonRpcClient
from xrpl.models import Ledger, Tx, AccountObjects, EscrowCreate, EscrowFinish
from xrpl.transaction import submit_and_wait
from xrpl.wallet import generate_faucet_wallet, Wallet
from xrpl.account import get_balance
from xrpl.utils import datetime_to_ripple_time


def initialize_clients():
    return {
        'testnet': JsonRpcClient("https://s.altnet.rippletest.net:51234/"),
        'mainnet': JsonRpcClient("https://xrplcluster.com/")
    }

def manual_wallet_fallback(label="wallet"):
    print(f"\n⚠️ Faucet failed for {label}. Please input a seed manually.")
    seed = input(f"🔐 Enter seed for {label}: ")
    return Wallet(seed=seed, sequence=0)

def get_wallet(client, label="wallet"):
    try:
        return generate_faucet_wallet(client)
    except Exception as e:
        print(f"\nWEE WOO! Faucet error for {label}: {e}")
        return manual_wallet_fallback(label)

def check_validated_transaction(client):
    try:
        ledger = client.request(Ledger(ledger_index="validated", transactions=True))
        print(f"\nLatest Ledger Index: {ledger.result['ledger']['ledger_index']}")
        if ledger.result['ledger']['transactions']:
            tx = client.request(Tx(transaction=ledger.result['ledger']['transactions'][0]))
            print("\nFirst Transaction in Ledger:")
            print(f"Hash: {tx.result.get('hash', 'N/A')}")
            print(f"Type: {tx.result.get('TransactionType', 'Unknown')}")
            if 'Account' in tx.result:
                print(f"From: {tx.result['Account']}")
            if 'Amount' in tx.result:
                print(f"Amount: {tx.result['Amount']} drops")
        else:
            print("\nNo transactions in the latest ledger")
    except Exception as e:
        print(f"\nError checking transactions: {str(e)}")

def demonstrate_escrow(client):
    try:
        print("\n=== Escrow Demonstration ===")
        wallet1 = get_wallet(client, "Wallet 1")
        wallet2 = get_wallet(client, "Wallet 2")

        print(f"\nWallet 1: {wallet1.address}")
        print(f"Wallet 2: {wallet2.address}")

        print("\nChecking balances...")
        print(f"Wallet 1: {get_balance(wallet1.address, client)} XRP")
        print(f"Wallet 2: {get_balance(wallet2.address, client)} XRP")

        escrow_delay = 30
        finish_time = datetime_to_ripple_time(datetime.now()) + escrow_delay
        print(f"\nCreating escrow with {escrow_delay}s release delay...")

        escrow_create = EscrowCreate(
            account=wallet1.address,
            destination=wallet2.address,
            amount="1000000",  # 1 XRP
            finish_after=finish_time,
        )
        create_response = submit_and_wait(escrow_create, client, wallet1)
        print(f"EscrowCreate Successful! Tx Hash: {create_response.result['hash']}")

        print(f"\nWaiting {escrow_delay + 5} seconds for escrow to become finishable...")
        sleep(escrow_delay + 5)

        print("\nFetching escrow object...")
        acct_objs = client.request(AccountObjects(account=wallet1.address)).result
        escrows = [obj for obj in acct_objs.get("account_objects", []) if obj["type"] == "escrow"]
        if not escrows:
            print("No escrow object found. Exiting.")
            return

        escrow_seq = escrows[0]["seq"]
        print(f"Found escrow with sequence: {escrow_seq}")

        escrow_finish = EscrowFinish(
            account=wallet1.address,
            owner=wallet1.address,
            offer_sequence=escrow_seq,
        )
        finish_response = submit_and_wait(escrow_finish, client, wallet1)
        print(f"EscrowFinish Successful! Tx Hash: {finish_response.result['hash']}")

        print("\nFinal balances:")
        print(f"Wallet 1: {get_balance(wallet1.address, client)} XRP")
        print(f"Wallet 2: {get_balance(wallet2.address, client)} XRP")

    except Exception as e:
        print(f"\nEscrow demonstration failed: {str(e)}")
        if "Insufficient balance" in str(e):
            print("Please ensure your test wallet has enough XRP (try the faucet again)")
