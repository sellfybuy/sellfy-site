import React, { useState, useMemo } from 'react';
import { useQuery } from '@tanstack/react-query';
import { base44 } from '@/api/base44Client';
import { Search } from 'lucide-react';
import { Input } from '@/components/ui/input';
import MenuHeader from '@/components/menu/MenuHeader';
import CategoryTabs from '@/components/menu/CategoryTabs';
import ProductCard from '@/components/menu/ProductCard';
import ProductModal from '@/components/menu/ProductModal';
import CartDrawer from '@/components/menu/CartDrawer';
import FloatingCartButton from '@/components/menu/FloatingCartButton';

export default function Menu() {
  const pathParts = window.location.pathname.split('/loja/');
  const slug = pathParts.length > 1 ? pathParts[1].split('/')[0] : '';

  const [activeCategory, setActiveCategory] = useState(null);
  const [search, setSearch] = useState('');
  const [selectedProduct, setSelectedProduct] = useState(null);
  const [cartOpen, setCartOpen] = useState(false);

  const { data: restaurants } = useQuery({
    queryKey: ['restaurant', slug],
    queryFn: () => base44.entities.Restaurant.filter({ slug, is_active: true }),
  });
  const restaurant = restaurants?.[0];

  const { data: categories = [] } = useQuery({
    queryKey: ['categories', restaurant?.id],
    queryFn: () => base44.entities.Category.filter({ restaurant_id: restaurant.id, is_active: true }, 'sort_order'),
    enabled: !!restaurant,
  });

  const { data: products = [] } = useQuery({
    queryKey: ['products', restaurant?.id],
    queryFn: () => base44.entities.Product.filter({ restaurant_id: restaurant.id, is_active: true }, 'sort_order'),
    enabled: !!restaurant,
  });

  const filteredProducts = useMemo(() => {
    let result = products;
    if (activeCategory) result = result.filter(p => p.category_id === activeCategory);
    if (search) {
      const q = search.toLowerCase();
      result = result.filter(p => p.name.toLowerCase().includes(q) || p.description?.toLowerCase().includes(q));
    }
    return result;
  }, [products, activeCategory, search]);

  const groupedProducts = useMemo(() => {
    if (activeCategory || search) return [{ id: 'all', name: '', products: filteredProducts }];
    return categories.map(cat => ({
      ...cat,
      products: filteredProducts.filter(p => p.category_id === cat.id)
    })).filter(g => g.products.length > 0);
  }, [filteredProducts, categories, activeCategory, search]);

  if (!restaurant) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-center">
          <div className="w-16 h-16 border-4 border-primary border-t-transparent rounded-full animate-spin mx-auto mb-4" />
          <p className="text-muted-foreground">Carregando cardápio...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-background pb-24">
      <MenuHeader restaurant={restaurant} />

      <div className="max-w-4xl mx-auto px-4 mt-6 mb-3">
        <div className="relative">
          <Search className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-muted-foreground" />
          <Input
            placeholder="Buscar no cardápio..."
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="pl-10 rounded-full bg-card"
          />
        </div>
      </div>

      <CategoryTabs categories={categories} activeId={activeCategory} onSelect={setActiveCategory} />

      <div className="max-w-4xl mx-auto px-4 py-6 space-y-8">
        {groupedProducts.map(group => (
          <div key={group.id}>
            {group.name && <h2 className="text-lg font-bold mb-3">{group.icon} {group.name}</h2>}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
              {group.products.map(product => (
                <ProductCard key={product.id} product={product} onClick={setSelectedProduct} />
              ))}
            </div>
          </div>
        ))}
        {filteredProducts.length === 0 && (
          <div className="text-center py-12 text-muted-foreground">
            <p className="text-lg">Nenhum produto encontrado</p>
          </div>
        )}
      </div>

      <ProductModal
        product={selectedProduct}
        open={!!selectedProduct}
        onClose={() => setSelectedProduct(null)}
      />

      <FloatingCartButton onClick={() => setCartOpen(true)} />
      <CartDrawer open={cartOpen} onClose={() => setCartOpen(false)} restaurantId={restaurant?.id} />
    </div>
  );
}
