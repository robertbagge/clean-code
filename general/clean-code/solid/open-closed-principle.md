# Open-Closed Principle (OCP)

## Overview

Software entities should be **open for extension**
but **closed for modification**. Add new features by extending behavior,
not by changing existing code.

## Core Concept

- Extend functionality through abstraction (interfaces, inheritance)
- Existing code remains untouched when adding features
- New requirements = new code, not modified code
- Use polymorphism to handle variations

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

## BAD — Modification for each new feature

```typescript
class DiscountCalculator {
  calculateDiscount(customer: Customer, product: Product): number {
    let discount = 0;
    
    // Adding new customer types requires modifying this method
    if (customer.type === 'regular') {
      discount = 0.05;
    } else if (customer.type === 'premium') {
      discount = 0.10;
    } else if (customer.type === 'vip') {
      discount = 0.20;
    }
    
    // Adding seasonal discounts requires modifying this method
    if (new Date().getMonth() === 11) { // December
      discount += 0.15;
    }
    
    // Adding category discounts requires modifying this method
    if (product.category === 'electronics') {
      discount += 0.05;
    }
    
    return Math.min(discount, 0.50);
  }
}
```

---

## GOOD — Extension through abstraction

```typescript
interface DiscountRule {
  calculate(customer: Customer, product: Product): number;
}

class CustomerDiscount implements DiscountRule {
  private rates = { regular: 0.05, premium: 0.10, vip: 0.20 };
  
  calculate(customer: Customer): number {
    return this.rates[customer.type] || 0;
  }
}

class SeasonalDiscount implements DiscountRule {
  calculate(): number {
    return new Date().getMonth() === 11 ? 0.15 : 0;
  }
}

class CategoryDiscount implements DiscountRule {
  calculate(customer: Customer, product: Product): number {
    return product.category === 'electronics' ? 0.05 : 0;
  }
}

class DiscountCalculator {
  private rules: DiscountRule[] = [];
  
  addRule(rule: DiscountRule): void {
    this.rules.push(rule);
  }
  
  calculateDiscount(customer: Customer, product: Product): number {
    const total = this.rules.reduce(
      (sum, rule) => sum + rule.calculate(customer, product),
      0
    );
    return Math.min(total, 0.50);
  }
}
```

---

## Anti-patterns to Avoid

1. **Switch statements that grow** with each new feature
2. **If-else chains** that require modification for new cases
3. **Concrete dependencies** that prevent extension

---

## Key Takeaways

- Use **interfaces/abstract classes** to define contracts
- Implement variations as **separate classes**
- **Compose behaviors** instead of modifying existing code
- New features = new classes implementing existing interfaces
- Existing tested code stays untouched and reliable
