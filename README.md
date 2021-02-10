
# -*- coding: utf-8 -*-  from odoo import fields, models, api, exceptions, _ from odoo.addons.base.models.decimal_precision import dp   class SaleSession(models.Model):     _name = 'sale.session'     _description = 'Session'     _inherit = 'mail.thread'      _sql_constraints = [         ('date_unique', 'unique(date)', 'Only 1/2 session per date is allowed!')     ]      @api.model     @api.depends('date')     def get_name(self):         self.name = self.date      @api.model     def get_default_warehouse(self):         return self.env['stock.warehouse'].search([('company_id', '=', self.env.company.id)], limit=1)         # return self.env.user.company_id.warehouse_ids[0].id if self.env.user.company_id.warehouse_ids else False      date = fields.Date(default=fields.Date.today(), readonly=True, states={'draft': [('readonly', False)]})     sale_orders = fields.One2many('sale.order', 'session', readonly=True, states={'draft': [('readonly', False)]},                                   string="Bons de commande")     name = fields.Char(string='Code', compute='get_name')     responsable = fields.Many2one('hr.employee', required=True, domain="[('responsable','=','True')]", readonly=True,                                   states={'draft': [('readonly', False)]})     logs = fields.One2many('sale.session.log', 'session', readonly=True, states={'draft': [('readonly', False)]})     state = fields.Selection([('draft', 'Brouillon'), ('except', 'Exception'), ('done', 'Fait')], 'Etat', readonly=True,                              copy=False, select=True, default='draft')     lines = fields.One2many('sale.session.line', 'session', string='Lignes de carburant')     client_lines = fields.One2many('sale.session.client_line', 'session', string="Lignes par client")     total_sales = fields.Float(digits_compute=dp.get_precision('Product Price'), compute='get_sales',                                string="Total des ventes")     fuel_sales = fields.Float(digits_compute=dp.get_precision('Product Price'), compute='get_sales',                               string="Ventes carburant")     other_sales = fields.Float(digits_compute=dp.get_precision('Product Price'), compute='get_sales',                                string="Autres ventes")     warehouse = fields.Many2one('stock.warehouse', required=True, ondelete='restrict', default=get_default_warehouse,                                 readonly=True, states={'draft': [('readonly', False)]}, string="Station Service")     note = fields.Text()      # @api.model     def action_confirm(self):         count = self.search_count([('state', '!=', 'done'), ('date', '&lt;', self.date)])         if count > 0:             raise exceptions.ValidationError('Une session antérieure est encore ouverte.')         for log in self.logs:             if log.diff &lt; 0:                 raise exceptions.ValidationError('La différence d\'index ne peut pas être une valeur negative.')         for line in self.lines:             if abs(line.diff_qty) > 1:                 self.state = 'except'                 return False         self.action_force()      @api.model     @api.onchange('logs')     def update_line(self):         for line in self.lines:             log_qty = 0             for log in self.logs:                 if log.pump.product.id == line.product.id:                     log_qty += log.diff             line.log_qty = log_qty      # @api.model     def action_force(self):         for order in self.sale_orders:             order.date_order = self.date             order.signal_workflow('order_confirm')             for picking in order.picking_ids:                 picking.action_confirm()                 picking.force_assign()                 picking.action_done()         for log in self.logs:             log.pump.counter = log.new_counter         self.state = 'done'      @api.model     @api.onchange('sale_orders')     def get_client_lines(self):         self.client_lines.unlink()         self.client_lines = []         partners = self.sale_orders.mapped('partner_id')         for partner in partners:             self.client_lines |= self.env['sale.session.client_line'].new({'partner': partner.id, 'session': self.id})      @api.depends('sale_orders')     def get_sales(self):         for record in self:             other = 0             total = 0             fuel = 0             for order in record.sale_orders:                 total += order.amount_total                 for line in order.order_line:                     if line.product_id.pumps:                         fuel += line.price_total                     else:                         other += line.price_total             record.total_sales = total             record.fuel_sales = fuel             record.other_sales = other      def action_import_data(self):         count = self.search_count([('state', '!=', 'done'), ('date', '&lt;', self.date)])         if count > 0:             raise exceptions.ValidationError('Une session antérieure est encore ouverte.')         self.logs.unlink()         self.write({'logs': [(0, 0, {'session': self.id, 'pump': x.id, 'old_counter': x.counter}) for x in                              self.env['stock.location.pump'].search(                                  [('location', 'child_of', self.warehouse.view_location_id.id)])]})         self.sale_orders.write({'session': False})         sale_orders = self.env['sale.order'].search([('session', '=', False), ('date_order', '>=', self.date)])         self.write({'sale_orders': [(4, x.id) for x in sale_orders]})         products = self.logs.mapped('pump.product')         self.lines.unlink()         self.write({'lines': [(0, 0, {'product': x.id, 'session': self.id}) for x in products]})      @api.model     @api.onchange('date')     def onchange_date(self):         self.lines.unlink()         self.logs.unlink()         self.sale_orders.unlink()   class SaleSessionLine(models.Model):     _name = 'sale.session.line'     _description = 'Ligne de carburant'      product = fields.Many2one('product.product', readonly=True, string="Article")     log_qty = fields.Float(digits_compute=dp.get_precision('Product UoS'), compute='get_log',                            string="Variance compteur")     sale_qty = fields.Float(digits_compute=dp.get_precision('Product UoS'), compute='get_sales',                             string="Ventes")     diff_qty = fields.Float(digits_compute=dp.get_precision('Product UoS'), compute='get_diff',                             string="Difference")     session = fields.Many2one('sale.session', ondelete='cascade')      # @api.model     @api.depends('session.sale_orders')     def get_sales(self):         for record in self:             sales = 0             for order in record.session.sale_orders:                 for line in order.order_line:                     if line.product_id.id == record.product.id:                         sales += line.product_uom_qty             record.sale_qty = sales      @api.depends('log_qty', 'sale_qty')     def get_diff(self):         for record in self:             record.diff_qty = record.log_qty - record.sale_qty      # @api.model     @api.depends('session.logs')     def get_log(self):         for record in self:             log_qty = 0             for log in record.session.logs:                 if log.pump.product.id == record.product.id:                     log_qty += log.diff             record.log_qty = log_qty   class SaleSessionClientLine(models.Model):     _name = 'sale.session.client_line'     _description = 'Ligne de vente client'      partner = fields.Many2one('res.partner', readonly=True, string="Client")     order_count = fields.Integer(compute='get_sales_and_count', string="Nombre de commandes")     sale_qty = fields.Float(digits_compute=dp.get_precision('Product UoS'), compute='get_sales_and_count',                             string="Ventes")     session = fields.Many2one('sale.session', ondelete='cascade')      # @api.model     @api.depends('session.sale_orders')     def get_sales_and_count(self):         for record in self:             sales = 0             count = 0             for order in record.session.sale_orders:                 if order.partner_id.id == record.partner.id:                     sales += order.amount_total                     count += 1             record.sale_qty = sales             record.order_count = count   class SaleSessionLog(models.Model):     _name = 'sale.session.log'     _description = 'Log'      session = fields.Many2one('sale.session', ondelete='cascade')     pump = fields.Many2one('stock.location.pump', ondelete='cascade', string="Pompe")     old_counter = fields.Float(digits_compute=dp.get_precision('Product UoS'), string="Ancien compteur")     new_counter = fields.Float(digits_compute=dp.get_precision('Product UoS'), required=True, default=0,                                string="Nouveau compteur")     diff = fields.Float(digits_compute=dp.get_precision('Product UoS'), compute='get_diff', string="Difference")     electric_counter = fields.Float(digits_compute=dp.get_precision('Product UoS'), compute='get_electric_counter',                                     string="Compteur électrique")      @api.depends('new_counter', 'old_counter')     def get_diff(self):         for record in self:             record.diff = record.new_counter - record.old_counter      @api.depends('new_counter')     def get_electric_counter(self):         for record in self:             record.electric_counter = record.new_counter + record.pump.electric_diff   class SaleOrder(models.Model):     _inherit = 'sale.order'      session = fields.Many2one('sale.session', ondelete='set null')     vehicle = fields.Many2one('fleet.vehicle', ondelete='set null', string="Véhicule")     vouchers_delivered = fields.One2many('sale.order.voucher', 'sale_order_delivered', readonly=True,                                          states={'draft': [('readonly', False)]}, string="Bons d'échange donnés")     vouchers_taken = fields.One2many('sale.order.voucher', 'sale_order_taken', readonly=True,                                      states={'draft': [('readonly', False)]}, string="Bons d'échange reçus")     delivery_order_ref = fields.Char(string="N° BL")      @api.model     def _amount_all(self):         res = super(SaleOrder, self)._amount_all()         for order in self:             for vd in order.vouchers_delivered:                 res[order.id]['amount_untaxed'] += vd.price_total                 res[order.id]['amount_total'] += vd.price_total             for vt in order.vouchers_taken:                 res[order.id]['amount_untaxed'] -= vt.price_total                 res[order.id]['amount_total'] -= vt.price_total         return res   class SaleOrderVoucher(models.Model):     _name = 'sale.order.voucher'      sale_order_delivered = fields.Many2one('sale.order', required=True, string="Bon de commande de délivrance")     sale_order_taken = fields.Many2one('sale.order', string="Bon de commande de consommation")     name = fields.Char('Code', required=True)     product = fields.Many2one('product.product', required=True, string="Article")     quantity = fields.Float(digits_compute=dp.get_precision('Product UoS'), required=True, string="Quantité")     uom = fields.Many2one('uom.uom', required=True, string="Unité de mesure")     price_unit = fields.Float(digits_compute=dp.get_precision('Product Price'), required=True, string="Prix unitaire")     price_total = fields.Float(digits_compute=dp.get_precision('Product Price'), compute='get_price_total',                                string="Prix total")     state = fields.Selection(         [('draft', 'draft'), ('delivered', 'Delivered'), ('collected', 'Collected'), ('done', 'Done')], 'Etat',         readonly=True, copy=False, select=True, default='draft')     taxes = fields.Many2many('account.tax', related='product.taxes_id', string='Taxes')      @api.model     @api.onchange('product')     def onchange_product(self):         if self.product:             self.uom = self.product.uom_id             if not self.sale_order_delivered.pricelist_id:                 warn_msg = _('You have to select a pricelist or a customer in the sales form !\n'                              'Please set one before choosing a product.')                 warn_msg = _("No Pricelist ! : ") + warn_msg + "\n\n"             else:                 price = self.sale_order_delivered.pricelist_id.with_context(                     uom=self.uom.id or self.product.uom_id.id, date=self.sale_order_delivered.date_order).price_get(                     prod_id=self.product.id, qty=self.quantity or 1.0, partner=self.sale_order_delivered.partner_id.id)[                     self.sale_order_delivered.pricelist_id.id]                 if price is False:                     warn_msg = _("Cannot find a pricelist line matching this product and quantity.\n"                                  "You have to change either the product, the quantity or the pricelist.")                      warn_msg += _("No valid pricelist line found ! :") + warn_msg + "\n\n"                 else:                     self.price_unit = price      @api.depends('quantity', 'price_unit')     def get_price_total(self):         for record in self:             record.price_total = record.price_unit * record.quantity  # class SaleMakeInvoice(models.TransientModel): #     _inherit = 'sale.make.invoice' # #     @api.model #     def make_invoices(self): #         result = super(SaleMakeInvoice, self).make_invoices() #         if self.grouped: #             orders = self.env['sale.order'].browse(self.env.context.get('active_ids', [])) #             for o in orders: #                 for i in o.invoice_ids: #                     i.merge_lines() #             return result
