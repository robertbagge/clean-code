# DRY Principle (Don't Repeat Yourself)

## Overview

Every piece of knowledge must have a single, unambiguous,
authoritative representation within a system.
Duplication leads to inconsistency and maintenance nightmares.

## Core Concept

- Single source of truth for logic and data
- Extract common patterns into reusable units
- Configuration and constants in one place
- Applies to code, schemas, documentation

---

## Scaffolding

```typescript
interface Product {
  price: number;
  category: string;
}

interface Customer {
  type: 'regular' | 'premium' | 'vip';
}
```

---

## BAD — Duplicated logic across methods

```typescript
class PricingService {
  calculateProductPrice(product: Product, customer: Customer): number {
    let price = product.price;
    
    // Discount logic duplicated
    if (customer.type === 'regular') {
      price *= 0.95;
    } else if (customer.type === 'premium') {
      price *= 0.90;
    } else if (customer.type === 'vip') {
      price *= 0.80;
    }
    
    // Tax calculation duplicated
    price *= 1.08; // 8% tax
    
    return price;
  }
  
  calculateCartTotal(products: Product[], customer: Customer): number {
    let total = 0;
    
    for (const product of products) {
      let price = product.price;
      
      // DUPLICATE: Same discount logic
      if (customer.type === 'regular') {
        price *= 0.95;
      } else if (customer.type === 'premium') {
        price *= 0.90;
      } else if (customer.type === 'vip') {
        price *= 0.80;
      }
      
      total += price;
    }
    
    // DUPLICATE: Same tax calculation
    total *= 1.08;
    
    return total;
  }
}
```

---

## GOOD — Single source of truth

```typescript
class PricingService {
  private readonly TAX_RATE = 0.08;
  private readonly DISCOUNT_RATES = {
    regular: 0.05,
    premium: 0.10,
    vip: 0.20
  };
  
  private calculateDiscount(
    price: number, 
    customerType: Customer['type']
  ): number {
    const rate = this.DISCOUNT_RATES[customerType] || 0;
    return price * rate;
  }
  
  private applyTax(amount: number): number {
    return amount * (1 + this.TAX_RATE);
  }
  
  private getDiscountedPrice(
    basePrice: number, 
    customerType: Customer['type']
  ): number {
    return basePrice - this.calculateDiscount(basePrice, customerType);
  }
  
  calculateProductPrice(product: Product, customer: Customer): number {
    const discountedPrice = this.getDiscountedPrice(product.price, customer.type);
    return this.applyTax(discountedPrice);
  }
  
  calculateCartTotal(products: Product[], customer: Customer): number {
    const subtotal = products.reduce(
      (sum, product) => sum + this.getDiscountedPrice(product.price, customer.type),
      0
    );
    return this.applyTax(subtotal);
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
