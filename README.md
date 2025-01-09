import hashlib
import time
import json
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import serialization, hashes
import unittest

class Block:
    def __init__(self, index, previous_hash, timestamp, data, hash, difficulty, reward):
        self.index = index
        self.previous_hash = previous_hash
        self.timestamp = timestamp
        self.data = data
        self.hash = hash
        self.difficulty = difficulty
        self.reward = reward

    def __repr__(self):
        return f"Block(index: {self.index}, previous_hash: {self.previous_hash}, timestamp: {self.timestamp}, data: {self.data}, hash: {self.hash}, difficulty: {self.difficulty}, reward: {self.reward})"

class SmartContract:
    def __init__(self, creator, participants, conditions, value):
        self.creator = creator
        self.participants = participants
        self.conditions = conditions  # Función que retorna True/False
        self.value = value
        self.active = True

    def execute(self, blockchain_state):
        if not self.active:
            return "Contrato ya ejecutado o cancelado."

        if self.conditions(blockchain_state):
            self._distribute_funds()
            self.active = False
            return "Contrato ejecutado con éxito."
        else:
            return "Condiciones no cumplidas, no se puede ejecutar."

    def cancel(self, requester):
        if requester == self.creator:
            self.active = False
            return "Contrato cancelado por el creador."

class Blockchain:
    def __init__(self):
        self.chain = []
        self.pending_transactions = []
        self.difficulty = 1  # Dificultad inicial
        self.votes = {}  # Almacena votos para gobernanza
        self.total_supply = 20000000  # Límite total de NODOS en circulación
        self.rewards = []  # Almacena recompensas por bloque minado
        self.current_reward = 100  # Recompensa inicial en NODOS
        self.blocks_mined = 0  # Contador de bloques minados
        self.create_genesis_block()
        self.generate_keys()

    def create_genesis_block(self):
        genesis_block = Block(0, "0", time.time(), "Genesis Block", "0", self.difficulty, self.current_reward)
        self.chain.append(genesis_block)

    def mine_block(self, index, previous_hash, timestamp, data, difficulty, miner):
        try:
            print(f"Minando bloque {index}...")
            nonce = 0
            start_time = time.time()  # Tiempo de inicio para medir eficiencia
            while True:
                hash_attempt = self.hash_block(previous_hash, timestamp, data, nonce)
                if hash_attempt.startswith("0" * difficulty):
                    elapsed_time = time.time() - start_time
                    self.rewards.append((miner, self.current_reward))  # Almacenar quién recibió la recompensa
                    print(f"Bloque minado con éxito después de {nonce} intentos en {elapsed_time:.2f} segundos. Recompensa de {self.current_reward} NODOS para {miner}.")
                    self.blocks_mined += 1
                    self.adjust_reward_and_difficulty()  # Ajustar recompensa y dificultad
                    return Block(index, previous_hash, timestamp, data, hash_attempt, difficulty, self.current_reward)
                nonce += 1
        except Exception as e:
            print(f"Error al minar el bloque: {e}")

    def adjust_reward_and_difficulty(self):
        if self.blocks_mined % 2000 == 0:
            self.current_reward //= 2  # Reducir recompensa a la mitad cada 2000 bloques
            print(f"Recompensa ajustada a: {self.current_reward} NODOS")
        if self.blocks_mined % 200000 == 0:
            self.difficulty += 1  # Aumentar dificultad cada 200000 bloques
            print(f"Dificultad ajustada a: {self.difficulty}")

    def hash_block(self, previous_hash, timestamp, data, nonce):
        block_string = f"{previous_hash}{timestamp}{data}{nonce}".encode()
        return hashlib.sha256(block_string).hexdigest()

    def add_transaction(self, transaction_data):
        try:
            if self.total_supply <= 0:
                raise Exception('No se pueden realizar más transacciones. Límite de suministro alcanzado.')
            signature = self.sign_transaction(transaction_data)
            self.pending_transactions.append((transaction_data, signature))
            print(f"Transacción añadida: {transaction_data} con firma: {signature}")
        except Exception as e:
            print(f"Error al añadir la transacción '{transaction_data}': {e}")

    def sign_transaction(self, transaction_data):
        try:
            message = transaction_data.encode()
            signature = self.private_key.sign(
                message,
                padding.PSS(
                    mgf=padding.MGF1(hashes.SHA256()),
                    salt_length=padding.PSS.MAX_LENGTH
                ),
                hashes.SHA256()
            )
            return signature.hex()
        except Exception as e:
            print(f"Error al firmar la transacción '{transaction_data}': {e}")
            return None

    def verify_transaction(self, transaction_data, signature):
        try:
            message = transaction_data.encode()
            signature_bytes = bytes.fromhex(signature)
            self.public_key.verify(
                signature_bytes,
                message,
                padding.PSS(
                    mgf=padding.MGF1(hashes.SHA256()),
                    salt_length=padding.PSS.MAX_LENGTH
                ),
                hashes.SHA256()
            )
            return True
        except Exception as e:
            print(f"Error al verificar la transacción '{transaction_data}': {e}")
            return False

    def vote(self, user, proposal):
        if user not in self.votes:
            self.votes[user] = proposal
            print(f"Usuario {user} ha votado por: {proposal}")
        else:
            print(f"Usuario {user} ya ha votado.")

    def record_transaction(self, transaction_data):
        with open('transactions_log.txt', 'a') as f:
            f.write(f"{transaction_data}\n")
            print(f"Transacción registrada: {transaction_data}")

    def save_chain(self):
        with open('blockchain_data.json', 'w') as f:
            json.dump(self.chain, f, default=lambda x: x.__dict__)

    def generate_keys(self):
        try:
            self.private_key = rsa.generate_private_key(
                public_exponent=65537,
                key_size=2048,
                backend=default_backend()
            )
            self.public_key = self.private_key.public_key()
            print("Claves generadas con éxito.")
        except Exception as e:
            print(f"Error al generar claves: {e}")

    def confidential_transaction(self, transaction_data):
        # Método básico para manejar transacciones confidenciales
        encrypted_data = self.encrypt_data(transaction_data)
        self.add_transaction(encrypted_data)
        print(f"Transacción confidencial añadida: {encrypted_data}")

    def encrypt_data(self, data):
        # Método básico para encriptar datos
        return f"encrypted({data})"  # Simulación de encriptación

    def governance_proposal(self, proposal):
        # Método para gestionar propuestas de gobernanza
        if proposal not in self.votes:
            self.votes[proposal] = 0
        self.votes[proposal] += 1
        print(f"Propuesta '{proposal}' votada.")

    def audit_log(self, transaction_data):
        # Método para registrar auditoría
        with open('audit_log.txt', 'a') as f:
            f.write(f"{time.ctime()}: {transaction_data}\n")
            print(f"Registro de auditoría: {transaction_data}")

# Crear la blockchain:
blockchain = Blockchain()

# Ejemplo de uso
blockchain.add_transaction("Transacción 1: Alice envía 10 tokens a Bob")
blockchain.add_transaction("Transacción 2: Bob envía 5 tokens a Charlie")
blockchain.add_transaction("Transacción 3: Charlie envía 2 tokens a Alice")
blockchain.add_transaction("Transacción 4: Alice envía 1 token a Bob")
blockchain.add_transaction("Transacción 5: Bob envía 3 tokens a David")
blockchain.add_transaction("Transacción 6: David envía 4 tokens a Emily")
blockchain.add_transaction("Transacción 7: Emily envía 5 tokens a Frank")

# Votar
blockchain.vote("Alice", "Propuesta 1")
blockchain.vote("Bob", "Propuesta 2")
blockchain.vote("Charlie", "Propuesta 1")

# Registrar transacciones
blockchain.record_transaction("Transacción 1: Alice envía 10 tokens a Bob")
blockchain.record_transaction("Transacción 2: Bob envía 5 tokens a Charlie")

# Minar bloques
blockchain.mine_block(1, blockchain.chain[-1].hash, time.time(), "Transacción 1: Alice envía 10 tokens a Bob", blockchain.difficulty, "Alice")
blockchain.mine_block(2, blockchain.chain[-1].hash, time.time(), "Transacción 2: Bob envía 5 tokens a Charlie", blockchain.difficulty, "Bob")
blockchain.mine_block(3, blockchain.chain[-1].hash, time.time(), "Transacción 3: Charlie envía 2 tokens a Alice", blockchain.difficulty, "Charlie")
blockchain.mine_block(4, blockchain.chain[-1].hash, time.time(), "Transacción 4: Alice envía 1 token a Bob", blockchain.difficulty, "Alice")
blockchain.mine_block(5, blockchain.chain[-1].hash, time.time(), "Transacción 5: Bob envía 3 tokens a David", blockchain.difficulty, "Bob")
blockchain.mine_block(6, blockchain.chain[-1].hash, time.time(), "Transacción 6: David envía 4 tokens a Emily", blockchain.difficulty, "David")
blockchain.mine_block(7, blockchain.chain[-1].hash, time.time(), "Transacción 7: Emily envía 5 tokens a Frank", blockchain.difficulty, "Emily")
