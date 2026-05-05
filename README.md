import os
import time
import json
import hashlib
import uuid
from typing import List


# -------------------- BLOCK --------------------
class Block:
    def __init__(self, index, timestamp, data, prev_hash):
        self.index = index
        self.timestamp = timestamp
        self.data = data
        self.prev_hash = prev_hash
        self.hash = self.calculate_hash()

    def calculate_hash(self):
        block_string = json.dumps({
            "index": self.index,
            "timestamp": self.timestamp,
            "data": self.data,
            "prev_hash": self.prev_hash
        }, sort_keys=True).encode()

        return hashlib.sha256(block_string).hexdigest()


# -------------------- BLOCKCHAIN --------------------
class Blockchain:
    def __init__(self):
        self.chain = [self.create_genesis_block()]

    def create_genesis_block(self):
        return Block(0, time.time(), {"msg": "Genesis Block"}, "0")

    def get_latest_block(self):
        return self.chain[-1]

    def add_block(self, data):
        prev_block = self.get_latest_block()
        new_block = Block(
            prev_block.index + 1,
            time.time(),
            data,
            prev_block.hash
        )
        self.chain.append(new_block)

    def is_valid(self):
        for i in range(1, len(self.chain)):
            curr = self.chain[i]
            prev = self.chain[i - 1]

            if curr.hash != curr.calculate_hash():
                return False
            if curr.prev_hash != prev.hash:
                return False

        return True


# -------------------- RESILIENT STORAGE --------------------
class ResilientStorage:
    def __init__(self, nodes: List[str], chunk_size=1024):
        self.nodes = nodes
        self.chunk_size = chunk_size
        self.blockchain = Blockchain()

        # Create node directories
        for node in nodes:
            os.makedirs(node, exist_ok=True)

    def store_file(self, file_path):
        file_id = str(uuid.uuid4())
        chunk_index = 0

        with open(file_path, "rb") as f:
            while True:
                chunk = f.read(self.chunk_size)
                if not chunk:
                    break

                node = self.nodes[chunk_index % len(self.nodes)]
                chunk_name = f"{file_id}_{chunk_index}"

                with open(os.path.join(node, chunk_name), "wb") as cf:
                    cf.write(chunk)

                chunk_index += 1

        # Record in blockchain
        self.blockchain.add_block({
            "action": "store",
            "file_id": file_id,
            "file_name": os.path.basename(file_path),
            "chunks": chunk_index,
            "time": time.time()
        })

        return file_id

    def retrieve_file(self, file_id, output_path):
        chunks = []

        # Collect chunks from nodes
        for node in self.nodes:
            for file in os.listdir(node):
                if file.startswith(file_id):
                    chunks.append(file)

        if not chunks:
            print("File not found!")
            return

        # Sort chunks
        chunks.sort(key=lambda x: int(x.split("_")[-1]))

        with open(output_path, "wb") as out:
            for chunk in chunks:
                for node in self.nodes:
                    path = os.path.join(node, chunk)
                    if os.path.exists(path):
                        with open(path, "rb") as cf:
                            out.write(cf.read())
                        break

        print("File retrieved successfully!")

    def verify_integrity(self):
        return self.blockchain.is_valid()


# -------------------- MAIN PROGRAM --------------------
def main():
    nodes = ["node1", "node2", "node3"]
    storage = ResilientStorage(nodes)

    while True:
        print("\n--- Blockchain Resilient Storage ---")
        print("1. Store File")
        print("2. Retrieve File")
        print("3. Verify Blockchain")
        print("4. Exit")

        choice = input("Enter choice: ")

        if choice == "1":
            path = input("Enter file path: ")
            if os.path.exists(path):
                file_id = storage.store_file(path)
                print("Stored! File ID:", file_id)
            else:
                print("File not found")

        elif choice == "2":
            file_id = input("Enter file ID: ")
            output = input("Enter output file path: ")
            storage.retrieve_file(file_id, output)

        elif choice == "3":
            if storage.verify_integrity():
                print("Blockchain is VALID")
            else:
                print("Blockchain is CORRUPTED")

        elif choice == "4":
            break

        else:
            print("Invalid choice")


if __name__ == "__main__":
    main()
