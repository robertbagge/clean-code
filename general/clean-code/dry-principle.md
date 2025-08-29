# DRY Principle (Don't Repeat Yourself)

## Overview

Every piece of knowledge must have a single, unambiguous, authoritative representation within a system. Duplication leads to inconsistency and maintenance nightmares.

## Core Concept

- Single source of truth for logic and data
- Extract common patterns into reusable units
- Configuration and constants in one place
- Applies to code, schemas, documentation

---

## Scaffolding

```typescript
// Types for pricing examples
interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
}

interface Customer {
  id: string;
  type: 'regular' | 'premium' | 'vip';
  memberSince: Date;
}
```

---

## BAD — Duplicated logic across methods

```typescript
class PricingService {
  calculateProductPrice(product: Product, customer: Customer): number {
    let finalPrice = product.price;
    
    // Discount logic duplicated
    if (customer.type === 'regular') {
      finalPrice = finalPrice * 0.95;
    } else if (customer.type === 'premium') {
      finalPrice = finalPrice * 0.90;
    } else if (customer.type === 'vip') {
      finalPrice = finalPrice * 0.80;
    }
    
    // Tax calculation duplicated
    const taxRate = 0.08;
    finalPrice = finalPrice * (1 + taxRate);
    
    // Shipping calculation duplicated
    if (finalPrice < 50) {
      finalPrice = finalPrice + 10;
    }
    
    return Math.round(finalPrice * 100) / 100;
  }
  
  calculateCartTotal(products: Product[], customer: Customer): number {
    let total = 0;
    
    for (const product of products) {
      let price = product.price;
      
      // DUPLICATE: Same discount logic
      if (customer.type === 'regular') {
        price = price * 0.95;
      } else if (customer.type === 'premium') {
        price = price * 0.90;
      } else if (customer.type === 'vip') {
        price = price * 0.80;
      }
      
      total += price;
    }
    
    // DUPLICATE: Same tax calculation
    const taxRate = 0.08;
    total = total * (1 + taxRate);
    
    // DUPLICATE: Same shipping logic
    if (total < 50) {
      total = total + 10;
    }
    
    return Math.round(total * 100) / 100;
  }
  
  generateInvoice(products: Product[], customer: Customer): string {
    let subtotal = 0;
    let invoice = 'INVOICE\n';
    
    for (const product of products) {
      let price = product.price;
      
      // DUPLICATE: Same discount logic again!
      if (customer.type === 'regular') {
        price = price * 0.95;
      } else if (customer.type === 'premium') {
        price = price * 0.90;
      } else if (customer.type === 'vip') {
        price = price * 0.80;
      }
      
      subtotal += price;
      invoice += `${product.name}: $${price}\n`;
    }
    
    // DUPLICATE: Same tax and shipping
    const taxRate = 0.08;
    const tax = subtotal * taxRate;
    const shipping = subtotal < 50 ? 10 : 0;
    
    invoice += `Tax: $${tax}\n`;
    invoice += `Shipping: $${shipping}\n`;
    invoice += `Total: $${subtotal + tax + shipping}\n`;
    
    return invoice;
  }
}
```

---

## GOOD — Single source of truth

```typescript
class PricingService {
  // Constants in one place
  private readonly TAX_RATE = 0.08;
  private readonly FREE_SHIPPING_THRESHOLD = 50;
  private readonly SHIPPING_COST = 10;
  
  private readonly DISCOUNT_RATES: Record<Customer['type'], number> = {
    regular: 0.05,
    premium: 0.10,
    vip: 0.20
  };
  
  // Reusable discount calculation
  private calculateDiscount(price: number, customerType: Customer['type']): number {
    const discountRate = this.DISCOUNT_RATES[customerType] || 0;
    return price * discountRate;
  }
  
  // Reusable tax calculation
  private calculateTax(amount: number): number {
    return amount * this.TAX_RATE;
  }
  
  // Reusable shipping calculation
  private calculateShipping(subtotal: number): number {
    return subtotal >= this.FREE_SHIPPING_THRESHOLD ? 0 : this.SHIPPING_COST;
  }
  
  // Reusable price calculation
  private applyCustomerPricing(basePrice: number, customerType: Customer['type']): number {
    const discount = this.calculateDiscount(basePrice, customerType);
    return basePrice - discount;
  }
  
  calculateProductPrice(product: Product, customer: Customer): number {
    const discountedPrice = this.applyCustomerPricing(product.price, customer.type);
    const tax = this.calculateTax(discountedPrice);
    const shipping = this.calculateShipping(discountedPrice);
    
    return Math.round((discountedPrice + tax + shipping) * 100) / 100;
  }
  
  calculateCartTotal(products: Product[], customer: Customer): number {
    const subtotal = products.reduce(
      (sum, product) => sum + this.applyCustomerPricing(product.price, customer.type),
      0
    );
    
    const tax = this.calculateTax(subtotal);
    const shipping = this.calculateShipping(subtotal);
    
    return Math.round((subtotal + tax + shipping) * 100) / 100;
  }
  
  generateInvoice(products: Product[], customer: Customer): string {
    const lines: string[] = ['INVOICE', '-------'];
    let subtotal = 0;
    
    for (const product of products) {
      const price = this.applyCustomerPricing(product.price, customer.type);
      subtotal += price;
      lines.push(`${product.name}: $${price.toFixed(2)}`);
    }
    
    const tax = this.calculateTax(subtotal);
    const shipping = this.calculateShipping(subtotal);
    const total = subtotal + tax + shipping;
    
    lines.push('-------');
    lines.push(`Subtotal: $${subtotal.toFixed(2)}`);
    lines.push(`Tax (${(this.TAX_RATE * 100).toFixed(0)}%): $${tax.toFixed(2)}`);
    lines.push(`Shipping: $${shipping.toFixed(2)}`);
    lines.push(`Total: $${total.toFixed(2)}`);
    
    return lines.join('\n');
  }
}
```

---

## Anti-patterns to Avoid

1. **Copy-paste programming** instead of extracting functions
2. **Magic numbers** scattered throughout code
3. **Duplicate validation logic** across endpoints

---

## Key Takeaways

- **Extract common logic** into well-named functions
- **Centralize configuration** and constants
- **Single source of truth** for business rules
- **Changes in one place** propagate everywhere
- **Reduces bugs** from inconsistent updates