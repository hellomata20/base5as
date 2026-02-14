# base5asimport time
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

# Put the pool address you want to watch here (Uniswap V3 pool on Base)
POOL = Web3.to_checksum_address("0x0000000000000000000000000000000000000000")

# keccak256("Swap(address,address,int256,int256,uint160,uint128,int24)")
SWAP_TOPIC = "0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67"


def to_int256(b: bytes) -> int:
    """Decode signed int256 from 32 bytes."""
    v = int.from_bytes(b, "big", signed=False)
    if v >= 2**255:
        v -= 2**256
    return v


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))
    if not w3.is_connected():
        raise RuntimeError("❌ Cannot connect to Base RPC")

    if POOL.lower() == "0x0000000000000000000000000000000000000000":
        raise RuntimeError("❌ Put a real Uniswap V3 pool address in POOL")

    print("✅ Connected to Base")
    print("Watching pool:", POOL)

    last = w3.eth.block_number
    print("Starting block:", last)

    while True:
        try:
            current = w3.eth.block_number

            if current > last:
                for b in range(last + 1, current + 1):
                    logs = w3.eth.get_logs(
                        {
                            "fromBlock": b,
                            "toBlock": b,
                            "address": POOL,        # only this pool
                            "topics": [SWAP_TOPIC], # only swaps
                        }
                    )

                    for log in logs:
                        data = log["data"]

                        # Swap event data layout:
                        # amount0 (int256) | amount1 (int256) | sqrtPriceX96 (uint160 padded) | liquidity (uint128) | tick (int24 padded)
                        raw = bytes.fromhex(data[2:])

                        amount0 = to_int256(raw[0:32])
                        amount1 = to_int256(raw[32:64])

                        tx = log["transactionHash"].hex()

                        print(f"\nBlock {b} swap | tx={tx}")
                        print("amount0:", amount0)
                        print("amount1:", amount1)

                last = current

            time.sleep(2)

        except KeyboardInterrupt:
            print("\nStopped.")
            return

        except Exception as e:
            print("Error:", str(e))
            time.sleep(3)


if __name__ == "__main__":
    main()
