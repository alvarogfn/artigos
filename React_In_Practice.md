## Controlled vs Uncontrolled

Uma decisão importante que afeta bastante na qualidade do seu código é determinar onde o estado de um componente deve residir, e esta decisão impacta diretamente a flexibilidade, reusabilidade e complexidade. 

Um exemplo clássico desta decisão é a implementação de um modal: deve ele gerenciar seu próprio estado de visibilidade ou delegar este controle ao componente pai?

### Componentes Controlados: Delegando o Controle

Em um componente controlado, o estado é gerenciado externamente pelo componente pai.

Esta abordagem oferece maior flexibilidade e permite que o componente pai coordene múltiplos estados relacionados.

Nessa abordagem, podemos dizer que a flexibilidade do Modal é alta, mas a 
responsabilidade sobre ele também é, pois, o componente pai precisa manter a lógica necessária para abrir e fechar o modal.

```tsx
// Exemplo de Modal controlado:
export function Modal({ isOpen, onChangeIsOpen, children }) {
  // O componente recebe o estado como prop e delega mudanças através de callbacks
  return (
    <dialog 
      open={isOpen}
      onClose={() => onChangeIsOpen(false)}
    >
      {children}
    </dialog>
   );
}

// Exemplo de uso:
export function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <Modal isOpen={isOpen} onChangeIsOpen={setIsOpen}>
        <p>Greetings, one and all!</p>
        <button onClick={() => setIsOpen(false)}>OK</button>
      </Modal>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
    </div>
  );
}
```
### Componentes Não Controlados: Autonomia Interna

Por outro lado, componentes não controlados mantêm e gerenciam seu próprio estado internamente. Esta abordagem reduz a complexidade do componente pai e é ideal quando o comportamento do componente pode ser encapsulado de forma independente.

```tsx
// Exemplo de modal não controlado:
export function Modal({ children, onOpenChange }) {
  const [isOpen, setIsOpen] = useState(false);
  
  const handleOpenChange = (newState) => {
    setIsOpen(newState);
    onOpenChange?.(newState); // Opcional: notifica o pai sobre mudanças
  };
  
  if (!isOpen) { 
    return (
      <button onClick={() => handleOpenChange(true)}>Open Modal</button>
    );
  }
  
  return (
    <dialog 
      open
      onClose={() => handleOpenChange(false)}
    >
      {children}
      <button onClick={() => handleOpenChange(false)}>OK</button>
    </dialog>
   );
}

// App.tsx
function App() {
  return (
    <Modal onOpenChange={(isOpen) => console.log('Modal state:', isOpen)}>
      <p>Greetings, one and all!</p>
    </Modal>
  );
}
```

Esta decisão de design influencia diretamente a manutenibilidade e reusabilidade do seu código.

Use componentes controlados quando:

* O componente pai precisa coordenar múltiplos estados relacionados
* Você precisa de controle preciso sobre o comportamento do componente
* O estado do componente precisa ser sincronizado com outras partes da aplicação

Use componentes não controlados quando:

* O comportamento do componente é independente do resto da aplicação
* Você quer reduzir a complexidade do componente pai
* O componente pode funcionar de forma autônoma


## Lidando com requisições HTTP

Essa é uma abordagem comum, e você já deve ter visto muitos exemplos de código que fazem chamadas HTTP dentro de um componente, variando detalhes, como o uso de `fetch` ou `axios`, ou a forma como o estado é gerenciado.

Você já deve ter visto exemplos de como refatorar esse código para um hook customizado, mas vamos fazer isso de novo.

Esse componente é relativamente simples, você tem 3 estados dentro do componente para representar o estado da 
requisição.

O useEffect é executado apenas uma vez, quando o componente é montado, e é a engrenagem principal para fazer a 
requisição toda funcionar.

Primeira abordagem:

```tsx
import { useEffect, useState } from "react";

function App() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<any>(null);
  const [data, setData] = useState<any>(null);
  
  useEffect(() => {
    const fetchData = async () => {
      setIsLoading(true);

      const response = await fetch("https://fakestoreapi.com/products");

      if (!response.ok) throw new Error("Failed to fetch data");

      const data = await response.json();

      setData(data);
      setIsLoading(false);
    };

    fetchData().catch((e) => setError(e));
  }, []);

  return (
    <div>
      {isLoading && <p>Loading...</p>}

      {error && <p>{error.message}</p>}

      {data && <pre>{JSON.stringify(data)}</pre>}
    </div>
  );
}

export default App;
```

### Separando a lógica de fetch em um hook

Segunda Abordagem 

Nessa abordagem, isolamos a lógica de fetch em um hook customizado, que é reutilizável em qualquer componente, 
mas também diminuímos a responsabilidade do componente, que agora só precisa lidar com a renderização do estado e a
requisição.

Mas isso ainda pode melhorar, o componente APP ainda precisa lidar com a lógica de fazer a requisição, apesar de não 
precisar lidar com o estado da requisição.

Segunda abordagem:
```tsx

// useFetch.tsx
import { useEffect, useState } from "react";

type useFetchParams<T> = {
  fetcher: () => Promise<T>;
  queryKey: string[];
};

export function useFetch<T>({ fetcher, queryKey }: useFetchParams<T>) {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<any>(null);
  const [data, setData] = useState<T | null>(null);

  const queryKeyString = JSON.stringify(queryKey);

  useEffect(() => {
    const fetchData = async () => {
      setIsLoading(true);

      const data = await fetcher();
      setData(data);

      setIsLoading(false);
    };

    fetchData().catch((e) => setError(e));
  }, [queryKeyString]);

  return { isLoading, error, data };
}
```


```tsx
// App.tsx
import { useFetch } from "./useFetch.tsx";

const ENDPOINT = "https://fakestoreapi.com/products";

function App() {
  const { isLoading, error, data } = useFetch({
    queryKey: [ENDPOINT],
    fetcher: async () => {
      const response = await fetch(ENDPOINT);
      if (!response.ok) throw new Error("Failed to fetch data");
      return response.json();
    },
  })

  return (
    <div>
      {isLoading && <p>Loading...</p>}

      {error && <p>{error.message}</p>}

      {data && <pre>{JSON.stringify(data)}</pre>}
    </div>
  );
}

export default App;
```

### Separando a Requisição em um Hook Customizado

Agora, é o que eu considero ideal como separação de responsabilidades. 
* O Hook customizado `useFetch` é responsável por lidar com a lógica de fazer a requisição; 
* O hook useProductFindAll é responsável por fazer a requisição de produtos;
* O componente App é responsável por lidar com a renderização dos produtos e o estado da requisição.

Terceira Abordagem
```tsx
// useProductFindAll.tsx
import { useFetch } from "./useFetch.tsx";


const ENDPOINT = "https://fakestoreapi.com/products";


async function fetchProductFindAll(params = {}) {
  const searchParams = new URLSearchParams(params);

  const response = await fetch(ENDPOINT + `?${searchParams.toString()}`);
  if (!response.ok) throw new Error("Failed to fetch data");
  return response.json();
}

export function useProductFindAll(params = {}) {
  return useFetch({
    queryKey: [ENDPOINT, params],
    fetcher: () => fetchProductFindAll(params)
  });
}
```

```tsx
// App.tsx
import { useProductFindAll } from "./useProductFindAll.tsx";

function App() {
  const { isLoading, error, data } = useProductFindAll({ limit: 6 })

  return (
    <div>
      {isLoading && <p>Loading...</p>}

      {error && <p>{error.message}</p>}

      {data && <pre>{JSON.stringify(data)}</pre>}
    </div>
  );
}

export default App;
```

Se você tem experiência ou conhece a biblioteca `react-query`, você pode perceber que o hook `useFetch` é muito 
parecido 
com o hook `useQuery` do `react-query`, e é verdade.

O `react-query` é uma biblioteca que abstrai a lógica de fazer requisições, e é uma ótima alternativa para lidar com 
estado "back-end" no front-end. A biblioteca é muito poderosa e tem muitos recursos, como cache, refetch, paginação.

Por isso, se você está lidando com muitas requisições no seu projeto, eu recomendo que você dê uma olhada na biblioteca
`react-query`.



## Lidando com Eventos 

Outro caso comum, e que você já deve ter visto muitas vezes, é a lógica de lidar com eventos dentro do próprio componente.

Apesar do código estar simples, a responsabilidade do código pode ser divida em partes.

O código de escutar o escape pode ser extraído, pois também seria útil para menus, selects, ou qualquer outro componente que precise ouvir eventos de teclado.

```tsx
import { useEffect, useState } from "react";

export function Modal({ isOpen, onChangeIsOpen }) {
  useEffect(() => {
    function handleClick(event: KeyboardEvent) {
      if (event.key === "Escape") {
        onChangeIsOpen(false);
      }
    }

    window.addEventListener("keydown", handleClick);

    return () => {
      window.removeEventListener("keydown", handleClick);
    };
  }, []);

  return (
    <div>
      <dialog open={isOpen}>
        <p>Greetings, one and all!</p>
        <button onClick={() => onChangeIsOpen(false)}>OK</button>
      </dialog>
      <button onClick={() => onChangeIsOpen(true)}>Abrir Modal</button>
    </div>
  );
}
```


### Extraindo a lógica de escutar o teclado em um hook

No hook, a ideia é que você possa passar o código da tecla e a função de 
callback, e o hook vai lidar com a lógica de escutar e remover o evento de teclado.

```tsx
// useKeyPress.tsx
import { useEffect } from "react";

type useKeyPressParams = {
  keyCode: string;
  callback: (event: KeyboardEvent) => void;
};

export function useKeyPress({ keyCode, callback }: useKeyPressParams) {
  useEffect(() => {
    function handleClick(event: KeyboardEvent) {
      if (event.key === keyCode) {
        callback(event);
      }
    }

    window.addEventListener("keydown", handleClick);

    return () => {
      window.removeEventListener("keydown", handleClick);
    };
  }, [keyCode, callback]);
}
```

Assim, o modal tem menos responsabilidades e o hook pode ser reutilizado em outros componentes.

```tsx
// Modal.tsx
import { useCallback, useState } from "react";
import { useKeyPress } from "../hooks/useKeyPress";

export function Modal({isOpen, onChangeIsOpen}) {
  useKeyPress({ keyCode: "Escape", callback: onChangeIsOpen });

  return (
    <div>
      <dialog open={isOpen}>
        <p>Greetings, one and all!</p>
        <button onClick={() => onChangeIsOpen(false)}>OK</button>
      </dialog>
      <button onClick={() => onChangeIsOpen(true)}>Abrir Modal</button>
    </div>
  );
}
```

### Isolando a lógica de abrir e fechar o modal em um hook

No exemplo acima, estamos delegando ao componente PAI para lidar com o estado do modal,
ainda, sim, dentro do próprio pai um estado deve ser criado e gerenciado.

É possível também encapsular a lógica de abrir e fechar o modal em um hook customizado,
assim o componente que usa o modal não precisa lidar com o estado do modal.

O código não está mais ou menos complexo, mas a responsabilidade de manter o estado
aberto ou fechado do modal foi delegada para o próprio hook.

```tsx
// hooks/useModal.tsx
import { PropsWithChildren, useCallback, useState } from "react";

export function useModal() {
  const [isOpen, setIsOpen] = useState(false);

  const closeModal = useCallback(() => {
    setIsOpen(false);
  }, []);

  const openModal = useCallback(() => {
    setIsOpen(true);
  }, []);

  const Modal = useCallback(
    ({ children }: PropsWithChildren) => (
      <dialog open={isOpen}>{children}</dialog>
    ),
    [isOpen],
  );

  return {
    openModal,
    closeModal,
    Modal,
  };
}
```


```tsx
// App.tsx
import { useModal } from "./hooks/useModal";

function App() {
  const { Modal, openModal, closeModal } = useModal();

  return (
    <div>
      <Modal>
        <p>Greetings, one and all!</p>
        <button onClick={closeModal}>OK</button>
      </Modal>

      <button onClick={openModal}></button>
    </div>
  );
}

export default App;
```

## Trabalhando com formulários

Um caso comum é a lógica de lidar com formulários, que pode ser extraída para um hook customizado.

```tsx
// form.tsx
import { FormEventHandler, useState } from "react";

function Forms() {
  const [title, setTitle] = useState("");
  const [price, setPrice] = useState("");
  const [description, setDescription] = useState("");
  const [image, setImage] = useState("");
  const [category, setCategory] = useState("");

  const [data, setData] = useState<any>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<any>(null);

  const handleSubmit: FormEventHandler<HTMLFormElement> = async (e) => {
    try {
      setIsLoading(true);

      e.preventDefault();

      const response = await fetch("https://fakestoreapi.com/products", {
        method: "POST",
        body: JSON.stringify({
          title,
          price,
          description,
          image,
          category,
        }),
      });

      const data = await response.json();

      setData(data);
    } catch (e) {
      setError(e);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form
      style={{ display: "flex", flexFlow: "column", gap: 10 }}
      onSubmit={handleSubmit}
    >
      <label htmlFor="name">Title:</label>
      <input
        type="text"
        id="title"
        name="title"
        value={title}
        onChange={({ target: { value } }) => setTitle(value)}
      />

      <label htmlFor="price">Price:</label>
      <input
        type="number"
        id="price"
        name="price"
        value={price}
        onChange={({ target: { value } }) => setPrice(value)}
      />

      <label htmlFor="description">Description:</label>
      <textarea
        id="description"
        name="description"
        value={description}
        onChange={({ target: { value } }) => setDescription(value)}
      />

      <label htmlFor="image">Image:</label>
      <input
        type="url"
        id="image"
        name="image"
        onChange={({ target: { value } }) => setImage(value)}
      />

      <label htmlFor="category">Category:</label>
      <select
        id="category"
        name="category"
        value={category}
        onChange={({ target: { value } }) => setCategory(value)}
      >
        <option value="electronics">electronics</option>
        <option value="jewelery">jewelery</option>
        <option value="men's clothing">men's clothing</option>
        <option value="women's clothing">women's clothing</option>
      </select>

      <button disabled={isLoading} type="submit">
        Send
      </button>

      <div>Error: {JSON.stringify(error)}</div>
      <div>Response: {JSON.stringify(data)}</div>
    </form>
  );
}

export default Forms;
```

No cerne, esse componente lida com a ideia de <mark style="background: #FFB8EBA6;">criar um produto</mark>[^1], tem um estado
para cada campo de formulário e para os estados do formulário.

A reescrita desse componente pode ser feita em etapas. Podemos aproveitar parte do que fizemos no exemplo de lidando com requisições:

### Extraindo estados da requisição

```tsx title=useMutate.tsx
// useMutate.tsx

import { useCallback, useState } from "react";

export type UseMutateParams<T, A> = {
  mutation: (args: A) => Promise<T>;
};

export function useMutate<T, A>({ mutation }: UseMutateParams<T, A>) {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<any>(null);
  const [data, setData] = useState<T | null>(null);

  const mutate = useCallback(
    async (args: A) => {
      try {
        setIsLoading(true);

        const data = await mutation(args);
        setData(data);
      } catch (e) {
        setError(e);
      } finally {
        setIsLoading(false);
      }
    },
    [mutation],
  );

  return {
    isLoading,
    error,
    data,
    mutate,
  };
}
```

```tsx
// Form.tsx
import { FormEventHandler, useState } from "react";
import { useMutate } from "../hooks/useMutate.tsx";

function Forms() {
  const [title, setTitle] = useState("");
  const [price, setPrice] = useState("");
  const [description, setDescription] = useState("");
  const [image, setImage] = useState("");
  const [category, setCategory] = useState("");

  const { mutate, data, isLoading, error } = useMutate({
    mutation: async (body) => {
      const response = await fetch("https://fakestoreapi.com/products", {
        method: "POST",
        body: JSON.stringify(body),
      });

      return response.json();
    },
  });

  const handleSubmit: FormEventHandler<HTMLFormElement> = async (e) => {
    e.preventDefault();

    await mutate(
      JSON.stringify({
        title,
        price,
        description,
        category,
        image,
      }),
    );
  };

  return (
    <form
      style={{ display: "flex", flexFlow: "column", gap: 10 }}
      onSubmit={handleSubmit}
    >
      <label htmlFor="name">Title:</label>
      <input
        type="text"
        id="title"
        name="title"
        value={title}
        onChange={({ target: { value } }) => setTitle(value)}
      />

      <label htmlFor="price">Price:</label>
      <input
        type="number"
        id="price"
        name="price"
        value={price}
        onChange={({ target: { value } }) => setPrice(value)}
      />

      <label htmlFor="description">Description:</label>
      <textarea
        id="description"
        name="description"
        value={description}
        onChange={({ target: { value } }) => setDescription(value)}
      />

      <label htmlFor="image">Image:</label>
      <input
        type="url"
        id="image"
        name="image"
        onChange={({ target: { value } }) => setImage(value)}
      />

      <label htmlFor="category">Category:</label>
      <select
        id="category"
        name="category"
        value={category}
        onChange={({ target: { value } }) => setCategory(value)}
      >
        <option value="electronics">electronics</option>
        <option value="jewelery">jewelery</option>
        <option value="men's clothing">men's clothing</option>
        <option value="women's clothing">women's clothing</option>
      </select>

      <button disabled={isLoading} type="submit">
        Send
      </button>

      <div>Error: {JSON.stringify(error)}</div>
      <div>Response: {JSON.stringify(data)}</div>
    </form>
  );
}

export default Forms;
```

### Descontrolando campos de formulário

```tsx
import { FormEventHandler } from "react";
import { useMutate } from "../hooks/useMutate.tsx";

function Forms() {
  const { mutate, data, isLoading, error } = useMutate({
    mutation: async (body) => {
      const response = await fetch("https://fakestoreapi.com/products", {
        method: "POST",
        body: JSON.stringify(body),
      });

      return response.json();
    },
  });

  const handleSubmit: FormEventHandler<HTMLFormElement> = async (event) => {
    event.preventDefault();

    const form = event.target as HTMLFormElement;
    const formData = new FormData(form);

    await mutate(
      JSON.stringify({
        title: formData.get("title"),
        price: formData.get("price"),
        description: formData.get("description"),
        category: formData.get("category"),
        image: formData.get("image"),
      }),
    );
  };

  return (
    <form
      style={{ display: "flex", flexFlow: "column", gap: 10 }}
      onSubmit={handleSubmit}
    >
      <label htmlFor="name">Title:</label>
      <input type="text" id="title" name="title" />

      <label htmlFor="price">Price:</label>
      <input type="number" id="price" name="price" />

      <label htmlFor="description">Description:</label>
      <textarea id="description" name="description" />

      <label htmlFor="image">Image:</label>
      <input type="url" id="image" name="image" />

      <label htmlFor="category">Category:</label>
      <select id="category" name="category">
        <option value="electronics">electronics</option>
        <option value="jewelery">jewelery</option>
        <option value="men's clothing">men's clothing</option>
        <option value="women's clothing">women's clothing</option>
      </select>

      <button disabled={isLoading} type="submit">
        Send
      </button>

      <div>Error: {JSON.stringify(error)}</div>
      <div>Response: {JSON.stringify(data)}</div>
    </form>
  );
}

export default Forms;
```


### Movendo a Requisição para o seu próprio Hook


```tsx
// useProductCreate.tsx
import { useMutate } from "./useMutate.tsx";

async function fetchProductCreate(body) {
  const response = await fetch("https://fakestoreapi.com/products", {
    method: "POST",
    body: JSON.stringify(body),
  });

  return response.json();
}

export function useProductCreate() {
  return useMutate({
    mutation: (formData: FormData) => {
      const payload = {
        title: formData.get("title"),
        price: formData.get("price"),
        description: formData.get("description"),
        category: formData.get("category"),
        image: formData.get("image"),
      };

      return fetchProductCreate(payload);
    },
  });
}
```

```tsx
// Form.tsx

import { FormEventHandler } from "react";
import { useProductCreate } from "../hooks/useProductCreate";

function Forms() {
  const { data, mutate, isLoading, error } = useProductCreate();

  const handleSubmit: FormEventHandler<HTMLFormElement> = async (event) => {
    event.preventDefault();

    const formData = new FormData(event.target as HTMLFormElement);

    await mutate(formData);
  };

  return (
    <form
      style={{ display: "flex", flexFlow: "column", gap: 10 }}
      onSubmit={handleSubmit}
    >
      <label htmlFor="name">Title:</label>
      <input type="text" id="title" name="title" />

      <label htmlFor="price">Price:</label>
      <input type="number" id="price" name="price" />

      <label htmlFor="description">Description:</label>
      <textarea id="description" name="description" />

      <label htmlFor="image">Image:</label>
      <input type="url" id="image" name="image" />

      <label htmlFor="category">Category:</label>
      <select id="category" name="category">
        <option value="electronics">electronics</option>
        <option value="jewelery">jewelery</option>
        <option value="men's clothing">men's clothing</option>
        <option value="women's clothing">women's clothing</option>
      </select>

      <button disabled={isLoading} type="submit">
        Send
      </button>

      <div>Error: {JSON.stringify(error)}</div>
      <div>Response: {JSON.stringify(data)}</div>
    </form>
  );
}

export default Forms;
```

[^1]: Teste
