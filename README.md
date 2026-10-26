const STORAGE_KEY = 'mi-efectivo-transactions-v1';

const form = document.getElementById('transactionForm');
const descriptionInput = document.getElementById('description');
const amountInput = document.getElementById('amount');
const typeInput = document.getElementById('type');
const balanceElement = document.getElementById('balance');
const incomeTotalElement = document.getElementById('incomeTotal');
const expenseTotalElement = document.getElementById('expenseTotal');
const savingsTotalElement = document.getElementById('savingsTotal');
const transactionListElement = document.getElementById('transactionList');
const resetButton = document.getElementById('resetButton');

let transactions = loadTransactions();

function loadTransactions() {
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    if (!stored) return [];
    const parsed = JSON.parse(stored);
    return Array.isArray(parsed) ? parsed : [];
  } catch (error) {
    console.error('No se pudieron cargar las transacciones:', error);
    return [];
  }
}

function saveTransactions() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(transactions));
}

function formatCurrency(value) {
  return new Intl.NumberFormat('es-MX', {
    style: 'currency',
    currency: 'MXN',
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(value);
}

function getSummary() {
  const income = transactions
    .filter((item) => item.type === 'income')
    .reduce((sum, item) => sum + Number(item.amount), 0);

  const expense = transactions
    .filter((item) => item.type === 'expense')
    .reduce((sum, item) => sum + Number(item.amount), 0);

  const balance = income - expense;
  const savings = Math.max(balance, 0);

  return { income, expense, balance, savings };
}

function renderSummary() {
  const { income, expense, balance, savings } = getSummary();

  balanceElement.textContent = formatCurrency(balance);
  incomeTotalElement.textContent = formatCurrency(income);
  expenseTotalElement.textContent = formatCurrency(expense);
  savingsTotalElement.textContent = formatCurrency(savings);
}

function renderTransactions() {
  if (!transactions.length) {
    transactionListElement.innerHTML = '<li class="empty-state">Aún no hay movimientos. Añade tu primera operación.</li>';
    return;
  }

  transactionListElement.innerHTML = transactions
    .slice()
    .reverse()
    .map((item) => {
      const sign = item.type === 'income' ? '+' : '-';
      const amountText = `${sign} ${formatCurrency(Number(item.amount))}`;
      const date = new Date(item.createdAt).toLocaleString('es-MX', {
        day: '2-digit',
        month: 'short',
        year: 'numeric',
        hour: '2-digit',
        minute: '2-digit',
      });

      return `
        <li class="transaction-item ${item.type}">
          <div class="transaction-main">
            <span class="transaction-name">${item.description}</span>
            <span class="transaction-meta">${item.type === 'income' ? 'Ingreso' : 'Gasto'} · ${date}</span>
          </div>
          <div class="transaction-amount">${amountText}</div>
          <button class="delete-button" type="button" data-id="${item.id}">Eliminar</button>
        </li>
      `;
    })
    .join('');
}

function updateDashboard() {
  renderSummary();
  renderTransactions();
}

function addTransaction(event) {
  event.preventDefault();

  const description = descriptionInput.value.trim();
  const amount = Number(amountInput.value);
  const type = typeInput.value;

  if (!description || !amount || amount <= 0) {
    window.alert('Ingresa una descripción válida y un monto mayor a cero.');
    return;
  }

  transactions.push({
    id: crypto.randomUUID(),
    description,
    amount: type === 'income' ? amount : amount,
    type,
    createdAt: new Date().toISOString(),
  });

  saveTransactions();
  form.reset();
  descriptionInput.focus();
  updateDashboard();
}

function removeTransaction(id) {
  transactions = transactions.filter((item) => item.id !== id);
  saveTransactions();
  updateDashboard();
}

form.addEventListener('submit', addTransaction);
transactionListElement.addEventListener('click', (event) => {
  const target = event.target;
  if (!(target instanceof HTMLElement)) return;

  const button = target.closest('.delete-button');
  if (!button) return;

  const { id } = button.dataset;
  if (id) {
    removeTransaction(id);
  }
});

resetButton.addEventListener('click', () => {
  const shouldReset = window.confirm('¿Deseas borrar todos los movimientos?');
  if (!shouldReset) return;

  transactions = [];
  saveTransactions();
  updateDashboard();
});

updateDashboard();

if (!window.crypto || !window.crypto.randomUUID) {
  const fallbackId = () => `tx-${Date.now()}-${Math.random().toString(16).slice(2)}`;
  const originalAdd = addTransaction;
  addTransaction = function (event) {
    originalAdd(event);
  };
}
