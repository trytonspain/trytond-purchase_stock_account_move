=================
Purchase Scenario
=================

Imports::

    >>> import datetime
    >>> from dateutil.relativedelta import relativedelta
    >>> from decimal import Decimal
    >>> from operator import attrgetter
    >>> from proteus import config, Model, Wizard
    >>> today = datetime.date.today()

Create database::

    >>> config = config.set_trytond()
    >>> config.pool.test = True

Install purchase::

    >>> Module = Model.get('ir.module.module')
    >>> purchase_module, = Module.find([
    ...     ('name', '=', 'purchase_stock_account_move')])
    >>> analytic_module, = Module.find([('name', '=', 'analytic_purchase')])
    >>> Module.install([purchase_module.id, analytic_module.id],
    ...     config.context)
    >>> Wizard('ir.module.module.install_upgrade').execute('upgrade')

Create company::

    >>> Currency = Model.get('currency.currency')
    >>> CurrencyRate = Model.get('currency.currency.rate')
    >>> currencies = Currency.find([('code', '=', 'EUR')])
    >>> if not currencies:
    ...     currency = Currency(name='Euro', symbol=u'€', code='EUR',
    ...         rounding=Decimal('0.01'), mon_grouping='[3, 3, 0]',
    ...         mon_decimal_point=',')
    ...     currency.save()
    ...     CurrencyRate(date=today + relativedelta(month=1, day=1),
    ...         rate=Decimal('1.0'), currency=currency).save()
    ... else:
    ...     currency, = currencies
    >>> Company = Model.get('company.company')
    >>> Party = Model.get('party.party')
    >>> company_config = Wizard('company.company.config')
    >>> company_config.execute('company')
    >>> company = company_config.form
    >>> party = Party(name='B2CK')
    >>> party.save()
    >>> company.party = party
    >>> company.currency = currency
    >>> company_config.execute('add')
    >>> company, = Company.find([])

Reload the context::

    >>> User = Model.get('res.user')
    >>> Group = Model.get('res.group')
    >>> config._context = User.get_preferences(True, config.context)

Create purchase user::

    >>> purchase_user = User()
    >>> purchase_user.name = 'Purchase'
    >>> purchase_user.login = 'purchase'
    >>> purchase_user.main_company = company
    >>> purchase_group, = Group.find([('name', '=', 'Purchase')])
    >>> purchase_user.groups.append(purchase_group)
    >>> purchase_user.save()

Create stock user::

    >>> stock_user = User()
    >>> stock_user.name = 'Stock'
    >>> stock_user.login = 'stock'
    >>> stock_user.main_company = company
    >>> stock_group, = Group.find([('name', '=', 'Stock')])
    >>> stock_user.groups.append(stock_group)
    >>> stock_user.save()

Create account user::

    >>> account_user = User()
    >>> account_user.name = 'Account'
    >>> account_user.login = 'account'
    >>> account_user.main_company = company
    >>> account_group, = Group.find([('name', '=', 'Account')])
    >>> account_user.groups.append(account_group)
    >>> account_user.save()

Create fiscal year::

    >>> FiscalYear = Model.get('account.fiscalyear')
    >>> Sequence = Model.get('ir.sequence')
    >>> SequenceStrict = Model.get('ir.sequence.strict')
    >>> fiscalyear = FiscalYear(name=str(today.year))
    >>> fiscalyear.start_date = today + relativedelta(month=1, day=1)
    >>> fiscalyear.end_date = today + relativedelta(month=12, day=31)
    >>> fiscalyear.company = company
    >>> post_move_seq = Sequence(name=str(today.year), code='account.move',
    ...     company=company)
    >>> post_move_seq.save()
    >>> fiscalyear.post_move_sequence = post_move_seq
    >>> invoice_seq = SequenceStrict(name=str(today.year),
    ...     code='account.invoice', company=company)
    >>> invoice_seq.save()
    >>> fiscalyear.out_invoice_sequence = invoice_seq
    >>> fiscalyear.in_invoice_sequence = invoice_seq
    >>> fiscalyear.out_credit_note_sequence = invoice_seq
    >>> fiscalyear.in_credit_note_sequence = invoice_seq
    >>> fiscalyear.save()
    >>> FiscalYear.create_period([fiscalyear.id], config.context)

Create chart of accounts::

    >>> AccountTemplate = Model.get('account.account.template')
    >>> Account = Model.get('account.account')
    >>> account_template, = AccountTemplate.find([('parent', '=', None)])
    >>> create_chart = Wizard('account.create_chart')
    >>> create_chart.execute('account')
    >>> create_chart.form.account_template = account_template
    >>> create_chart.form.company = company
    >>> create_chart.execute('create_account')
    >>> receivable, = Account.find([
    ...         ('kind', '=', 'receivable'),
    ...         ('company', '=', company.id),
    ...         ])
    >>> payable, = Account.find([
    ...         ('kind', '=', 'payable'),
    ...         ('company', '=', company.id),
    ...         ])
    >>> expense, = Account.find([
    ...         ('kind', '=', 'expense'),
    ...         ('company', '=', company.id),
    ...         ])
    >>> expense.code = 'E1'
    >>> expense.save()
    >>> expense2 = Account()
    >>> expense2.code = 'E2'
    >>> expense2.name = 'Second expense'
    >>> expense2.type = expense.type
    >>> expense2.kind = 'expense'
    >>> expense2.parent = expense.parent
    >>> expense2.save()
    >>> pending_payable = Account()
    >>> pending_payable.code = 'PR'
    >>> pending_payable.name = 'Pending payable'
    >>> pending_payable.type = payable.type
    >>> pending_payable.kind = 'payable'
    >>> pending_payable.reconcile = True
    >>> pending_payable.parent = payable.parent
    >>> pending_payable.save()
    >>> revenue, = Account.find([
    ...         ('kind', '=', 'revenue'),
    ...         ('company', '=', company.id),
    ...         ])
    >>> create_chart.form.account_receivable = receivable
    >>> create_chart.form.account_payable = payable
    >>> create_chart.execute('create_properties')

Create analytic accounts::

    >>> AnalyticAccount = Model.get('analytic_account.account')
    >>> root = AnalyticAccount(type='root', name='Root')
    >>> root.save()
    >>> analytic_account = AnalyticAccount(root=root, parent=root,
    ...     name='Analytic', display_balance='debit-credit')
    >>> analytic_account.save()

Configure purchase to track pending_payables in accounting::

    >>> PurchaseConfig = Model.get('purchase.configuration')
    >>> purchase_config = PurchaseConfig(1)
    >>> purchase_config.purchase_shipment_method = 'order'
    >>> purchase_config.purchase_invoice_method = 'shipment'
    >>> purchase_config.pending_invoice_account = pending_payable
    >>> purchase_config.save()

Create parties::

    >>> Party = Model.get('party.party')
    >>> supplier = Party(name='Supplier')
    >>> supplier.save()
    >>> customer = Party(name='Customer')
    >>> customer.save()

Create category::

    >>> ProductCategory = Model.get('product.category')
    >>> category = ProductCategory(name='Category')
    >>> category.save()

Create products::

    >>> ProductUom = Model.get('product.uom')
    >>> unit, = ProductUom.find([('name', '=', 'Unit')])
    >>> ProductTemplate = Model.get('product.template')
    >>> Product = Model.get('product.product')
    >>> product1 = Product()
    >>> template1 = ProductTemplate()
    >>> template1.name = 'product'
    >>> template1.category = category
    >>> template1.default_uom = unit
    >>> template1.type = 'goods'
    >>> template1.purchasable = True
    >>> template1.salable = True
    >>> template1.list_price = Decimal('20')
    >>> template1.cost_price = Decimal('15')
    >>> template1.cost_price_method = 'fixed'
    >>> template1.account_expense = expense
    >>> template1.account_revenue = revenue
    >>> template1.save()
    >>> product1.template = template1
    >>> product1.save()
    >>> template2 = ProductTemplate()
    >>> template2.name = 'product'
    >>> template2.category = category
    >>> template2.default_uom = unit
    >>> template2.type = 'goods'
    >>> template2.purchasable = True
    >>> template2.salable = True
    >>> template2.list_price = Decimal('40')
    >>> template2.cost_price = Decimal('25')
    >>> template2.cost_price_method = 'fixed'
    >>> template2.account_expense = expense2
    >>> template2.account_revenue = revenue
    >>> template2.save()
    >>> product2 = Product()
    >>> product2.template = template2
    >>> product2.save()
    >>> service_product = Product()
    >>> service_template = ProductTemplate()
    >>> service_template.name = 'product'
    >>> service_template.category = category
    >>> service_template.default_uom = unit
    >>> service_template.type = 'service'
    >>> service_template.purchasable = True
    >>> service_template.salable = True
    >>> service_template.list_price = Decimal('20')
    >>> service_template.cost_price = Decimal('15')
    >>> service_template.cost_price_method = 'fixed'
    >>> service_template.account_expense = expense
    >>> service_template.account_revenue = revenue
    >>> service_template.save()
    >>> service_product.template = service_template
    >>> service_product.save()

Create payment term::

    >>> PaymentTerm = Model.get('account.invoice.payment_term')
    >>> PaymentTermLine = Model.get('account.invoice.payment_term.line')
    >>> payment_term = PaymentTerm(name='Direct')
    >>> payment_term_line = PaymentTermLine(type='remainder', days=0)
    >>> payment_term.lines.append(payment_term_line)
    >>> payment_term.save()

Purchase products::

    >>> config.user = purchase_user.id
    >>> Purchase = Model.get('purchase.purchase')
    >>> PurchaseLine = Model.get('purchase.line')
    >>> AnalyticSelection = Model.get('analytic_account.account.selection')
    >>> purchase = Purchase()
    >>> purchase.party = supplier
    >>> purchase.payment_term = payment_term
    >>> purchase_line = PurchaseLine()
    >>> purchase.lines.append(purchase_line)
    >>> purchase_line.product = product1
    >>> purchase_line.quantity = 20.0
    >>> analytic_selection = AnalyticSelection()
    >>> analytic_selection.accounts.append(analytic_account)
    >>> analytic_selection.save()
    >>> purchase_line.analytic_accounts = analytic_selection
    >>> purchase_line = PurchaseLine()
    >>> purchase.lines.append(purchase_line)
    >>> purchase_line.type = 'comment'
    >>> purchase_line.description = 'Comment'
    >>> purchase_line = PurchaseLine()
    >>> purchase.lines.append(purchase_line)
    >>> purchase_line.product = product2
    >>> purchase_line.quantity = 20.0
    >>> analytic_account, = AnalyticAccount.find([('type', '=', 'normal')])
    >>> analytic_selection = AnalyticSelection()
    >>> analytic_selection.accounts.append(analytic_account)
    >>> analytic_selection.save()
    >>> purchase_line.analytic_accounts = analytic_selection
    >>> purchase.save()
    >>> purchase.click('quote')
    >>> purchase.click('confirm')
    >>> purchase.click('process')
    >>> purchase.state
    u'processing'
    >>> purchase.reload()
    >>> len(purchase.moves), len(purchase.shipment_returns), len(purchase.invoices)
    (2, 0, 0)
    >>> config.user = account_user.id
    >>> AccountMoveLine = Model.get('account.move.line')
    >>> moves = AccountMoveLine.find([
    ...     ('account', '=', pending_payable.id)
    ...     ])
    >>> len(moves) == 0
    True
    >>> analytic_account.reload()
    >>> analytic_account.debit == Decimal('0.0')
    True

Validate Shipments::

    >>> moves = purchase.moves
    >>> config.user = stock_user.id
    >>> Move = Model.get('stock.move')
    >>> ShipmentIn = Model.get('stock.shipment.in')
    >>> shipment = ShipmentIn()
    >>> shipment.supplier = supplier
    >>> for move in moves:
    ...     incoming_move = Move(id=move.id)
    ...     incoming_move.quantity = 15.0
    ...     shipment.incoming_moves.append(incoming_move)
    >>> shipment.save()
    >>> ShipmentIn.receive([shipment.id], config.context)
    >>> ShipmentIn.done([shipment.id], config.context)
    >>> config.user = account_user.id
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account', '=', pending_payable.id),
    ...     ])
    >>> len(account_moves) == 1
    True
    >>> sum([a.credit for a in account_moves]) == Decimal('600.0')
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account.code', '=', 'E1'),
    ...     ])
    >>> len(account_moves) == 1
    True
    >>> sum([a.debit for a in account_moves]) == Decimal('225.0')
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account.code', '=', 'E2'),
    ...     ])
    >>> len(account_moves) == 1
    True
    >>> sum([a.debit for a in account_moves]) == Decimal('375.0')
    True
    >>> analytic_account.reload()
    >>> analytic_account.debit == Decimal('600.0')
    True
    >>> config.user = purchase_user.id
    >>> purchase.reload()
    >>> moves = purchase.moves.find([('state', '=', 'draft')])
    >>> config.user = stock_user.id
    >>> shipment = ShipmentIn()
    >>> shipment.supplier = supplier
    >>> for move in moves:
    ...     incoming_move = Move(id=move.id)
    ...     shipment.incoming_moves.append(incoming_move)
    >>> shipment.save()
    >>> ShipmentIn.receive([shipment.id], config.context)
    >>> ShipmentIn.done([shipment.id], config.context)
    >>> config.user = account_user.id
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account', '=', pending_payable.id),
    ...     ])
    >>> len(account_moves) == 2
    True
    >>> sum([a.credit for a in account_moves]) == Decimal('800.0')
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account.code', '=', 'E1'),
    ...     ])
    >>> len(account_moves) == 2
    True
    >>> sum([a.debit for a in account_moves]) == Decimal('300.0')
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account.code', '=', 'E2'),
    ...     ])
    >>> len(account_moves) == 2
    True
    >>> sum([a.debit for a in account_moves]) == Decimal('500.0')
    True
    >>> analytic_account.reload()
    >>> analytic_account.debit == Decimal('800.0')
    True

Open supplier invoices::

    >>> config.user = purchase_user.id
    >>> purchase.reload()
    >>> Invoice = Model.get('account.invoice')
    >>> invoice1, invoice2 = purchase.invoices
    >>> config.user = account_user.id
    >>> invoice1.invoice_date = today
    >>> invoice1.save()
    >>> Invoice.post([invoice1.id], config.context)
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account', '=', pending_payable.id),
    ...     ('reconciliation', '=', None),
    ...     ])
    >>> line, = account_moves
    >>> line.credit == Decimal('200.0')
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('account.code', '=', 'E1'),
    ...     ])
    >>> sum([a.debit - a.credit for a in account_moves]) == Decimal('300.0')
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('account.code', '=', 'E2'),
    ...     ])
    >>> sum([a.debit - a.credit for a in account_moves]) == Decimal('500.0')
    True
    >>> analytic_account.reload()
    >>> analytic_account.balance == Decimal('800.0')
    True
    >>> invoice2.invoice_date = today
    >>> invoice2.save()
    >>> Invoice.post([invoice2.id], config.context)
    >>> account_moves = AccountMoveLine.find([
    ...     ('origin', '=', 'purchase.purchase,' + str(purchase.id)),
    ...     ('account', '=', pending_payable.id),
    ...     ])
    >>> sum(l.debit - l.credit for l in account_moves) == Decimal('0.0')
    True
    >>> all(a.reconciliation is not None for a in account_moves)
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('account.code', '=', 'E1'),
    ...     ])
    >>> sum([a.debit - a.credit for a in account_moves]) == Decimal('300.0')
    True
    >>> account_moves = AccountMoveLine.find([
    ...     ('account.code', '=', 'E2'),
    ...     ])
    >>> sum([a.debit - a.credit for a in account_moves]) == Decimal('500.0')
    True
    >>> analytic_account.reload()
    >>> analytic_account.balance == Decimal('800.0')
    True