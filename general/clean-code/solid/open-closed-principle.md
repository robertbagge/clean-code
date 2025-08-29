# Open-Closed Principle (OCP)

## Overview

Software entities should be **open for extension** but **closed for modification**. Add new features by extending behavior, not by changing existing code.

## Core Concept

- Extend functionality through abstraction (interfaces, inheritance)
- Existing code remains untouched when adding features
- New requirements = new code, not modified code
- Use polymorphism to handle variations

---

## Scaffolding

```typescript
// Types for discount calculation examples
interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
}

interface Customer {
  id: string;
  type: 'regular' | 'premium' | 'vip';
  joinDate: Date;
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
    const month = new Date().getMonth();
    if (month === 11) { // December
      discount += 0.15;
    } else if (month === 6) { // July
      discount += 0.10;
    }
    
    // Adding category-based discounts requires modifying this method
    if (product.category === 'electronics') {
      discount += 0.05;
    } else if (product.category === 'books') {
      discount += 0.03;
    }
    
    return Math.min(discount, 0.50); // Cap at 50%
  }
}
```

---

## GOOD — Extension through abstraction

```typescript
// Define abstraction for discount rules
interface DiscountRule {
  calculate(customer: Customer, product: Product): number;
}

// Extend by adding new rule classes
class CustomerTypeDiscount implements DiscountRule {
  private readonly discounts = {
    regular: 0.05,
    premium: 0.10,
    vip: 0.20
  };
  
  calculate(customer: Customer): number {
    return this.discounts[customer.type] || 0;
  }
}

class SeasonalDiscount implements DiscountRule {
  private readonly monthlyDiscounts = {
    6: 0.10,  // July
    11: 0.15  // December
  };
  
  calculate(): number {
    const month = new Date().getMonth();
    return this.monthlyDiscounts[month] || 0;
  }
}

class CategoryDiscount implements DiscountRule {
  private readonly categoryDiscounts = {
    electronics: 0.05,
    books: 0.03
  };
  
  calculate(customer: Customer, product: Product): number {
    return this.categoryDiscounts[product.category] || 0;
  }
}

// Calculator is closed for modification, open for extension
class DiscountCalculator {
  private rules: DiscountRule[] = [];
  
  addRule(rule: DiscountRule): void {
    this.rules.push(rule);
  }
  
  calculateDiscount(customer: Customer, product: Product): number {
    const totalDiscount = this.rules.reduce(
      (sum, rule) => sum + rule.calculate(customer, product),
      0
    );
    return Math.min(totalDiscount, 0.50); // Cap at 50%
  }
}

// Usage - extend without modifying existing code
const calculator = new DiscountCalculator();
calculator.addRule(new CustomerTypeDiscount());
calculator.addRule(new SeasonalDiscount());
calculator.addRule(new CategoryDiscount());
// Easy to add new rules without touching calculator or existing rules
// calculator.addRule(new BlackFridayDiscount());
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