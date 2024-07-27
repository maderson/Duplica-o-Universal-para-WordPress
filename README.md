# Função de duplicação universal para WordPress

Este repositório contém uma função para WordPress que permite duplicar qualquer tipo de conteúdo, incluindo posts, páginas e tipos de postagens personalizadas. Ideal para administradores e editores que precisam criar rascunhos ou cópias de conteúdo existente, mantendo todas as características originais, como metadados, categorias, tags, comentários e taxonomias.

## Funcionalidades

- **Duplicação completo e flexível:** Permite duplicar não apenas posts e páginas padrão, mas também qualquer tipo de post personalizado, preservando todos os detalhes, incluindo:
  - Conteúdo e título
  - Metadados e configurações de SEO
  - Categorias e tags
  - Comentários e suas configurações
  - Taxonomias personalizadas

- **Interface intuitiva:** Adiciona uma opção "Duplicar" na lista de ações para cada item, tornando o processo simples e direto para o usuário.

- **Compatibilidade total:** Suporta todos os tipos de postagens personalizadas, tornando-o uma ferramenta essencial para sites com estruturas de conteúdo complexas.

## Instruções de instalação

1. **Adicione o código:**
   - Copie o código fornecido e cole-o no arquivo `functions.php` do seu tema WordPress ativo. Para acessar este arquivo, vá até `Aparência > Editor de Temas` no painel administrativo do WordPress e selecione `functions.php` na lista de arquivos do tema.

2. **Salve as alterações:**
   - Após adicionar o código, salve o arquivo `functions.php`.

3. **Uso:**
   - Navegue até a lista de qualquer tipo de postagem, passe o mouse sobre o item que deseja duplicar e clique na opção "Duplicar". Uma nova cópia de rascunho será criada e pronta para edição.

## Requisitos

- WordPress 4.0 ou superior
- PHP 5.6 ou superior

## Contribuição

Contribuições são bem-vindas! Sinta-se à vontade para abrir issues ou enviar pull requests para melhorias e novas funcionalidades.

```
// Adiciona uma ação de duplicação ao menu de ações de posts e páginas
function custom_duplicate_post_link($actions, $post) {
    // Verifica se o usuário tem permissão para editar posts
    if (current_user_can('edit_posts')) {
        // Adiciona o link de duplicação à lista de ações
        $actions['duplicate'] = '<a href="' . wp_nonce_url('admin.php?action=custom_duplicate_post_as_draft&post=' . $post->ID, basename(__FILE__), 'duplicate_nonce' ) . '" title="' . __('Duplicar este item') . '" rel="permalink">' . __('Duplicar') . '</a>';
    }
    return $actions;
}
// Aplica o filtro para posts e páginas na lista de posts do admin
add_filter('page_row_actions', 'custom_duplicate_post_link', 10, 2);
add_filter('post_row_actions', 'custom_duplicate_post_link', 10, 2);

/**
 * Duplica o post selecionado como rascunho
 */
function custom_duplicate_post_as_draft() {
    global $wpdb;

    // Verifica se um post foi fornecido para duplicação
    if (!(isset($_GET['post']) || isset($_POST['post']) || (isset($_REQUEST['action']) && 'custom_duplicate_post_as_draft' == $_REQUEST['action']))) {
        wp_die('Nenhum post para duplicar foi fornecido!');
    }

    // Verifica o nonce de segurança
    $nonce = isset($_REQUEST['duplicate_nonce']) ? $_REQUEST['duplicate_nonce'] : '';
    if (!wp_verify_nonce($nonce, basename(__FILE__))) {
        return;
    }

    // Obtém o ID do post a ser duplicado
    $post_id = isset($_GET['post']) ? $_GET['post'] : $_POST['post'];
    $post = get_post($post_id);

    // Obtém o ID do usuário atual como autor do novo post
    $current_user = wp_get_current_user();
    $new_post_author = $current_user->ID;

    // Se o post existe, prepara os dados para duplicação
    if (isset($post) && $post != null) {
        // Prepara os argumentos para o novo post
        $args = array(
            'comment_status' => $post->comment_status,
            'ping_status'    => $post->ping_status,
            'post_author'    => $new_post_author,
            'post_content'   => $post->post_content,
            'post_excerpt'   => $post->post_excerpt,
            'post_name'      => $post->post_name,
            'post_parent'    => $post->post_parent,
            'post_password'  => $post->post_password,
            'post_status'    => 'draft', // Define o status do post como rascunho
            'post_title'     => $post->post_title,
            'post_type'      => $post->post_type,
            'to_ping'        => $post->to_ping,
            'menu_order'     => $post->menu_order
        );

        // Insere o novo post no banco de dados
        $new_post_id = wp_insert_post($args);

        // Duplica as categorias do post original
        $post_categories = wp_get_post_categories($post_id);
        foreach ($post_categories as $category_id) {
            wp_set_post_categories($new_post_id, $category_id, true);
        }

        // Duplica as tags do post original
        $post_tags = wp_get_post_tags($post_id);
        foreach ($post_tags as $tag) {
            wp_add_post_tags($new_post_id, $tag->name);
        }

        // Duplica as taxonomias do post original
        $taxonomies = get_object_taxonomies($post->post_type);
        foreach ($taxonomies as $taxonomy) {
            $post_terms = wp_get_object_terms($post_id, $taxonomy, array('fields' => 'slugs'));
            wp_set_object_terms($new_post_id, $post_terms, $taxonomy, false);
        }

        // Duplica os comentários do post original
        $comments = get_comments(array('post_id' => $post_id));
        foreach ($comments as $comment) {
            $commentdata = array(
                'comment_post_ID' => $new_post_id,
                'comment_author' => $comment->comment_author,
                'comment_author_email' => $comment->comment_author_email,
                'comment_author_url' => $comment->comment_author_url,
                'comment_content' => $comment->comment_content,
                'comment_type' => $comment->comment_type,
                'comment_parent' => $comment->comment_parent,
                'user_id' => $comment->user_id,
                'comment_author_IP' => $comment->comment_author_IP,
                'comment_agent' => $comment->comment_agent,
                'comment_date' => $comment->comment_date,
                'comment_approved' => $comment->comment_approved,
            );
            wp_insert_comment($commentdata);
        }

        // Duplica os metadados do post original
        $post_meta_infos = $wpdb->get_results("SELECT meta_key, meta_value FROM $wpdb->postmeta WHERE post_id=$post_id");

        if (count($post_meta_infos) != 0) {
            $sql_query = "INSERT INTO $wpdb->postmeta (post_id, meta_key, meta_value) ";
            foreach ($post_meta_infos as $meta_info) {
                $meta_key = $meta_info->meta_key;
                $meta_value = addslashes($meta_info->meta_value);
                $sql_query_sel[] = "SELECT $new_post_id, '$meta_key', '$meta_value'";
            }
            $sql_query .= implode(" UNION ALL ", $sql_query_sel);
            $wpdb->query($sql_query);
        }

        // Redireciona para a tela de edição do novo post
        wp_redirect(admin_url('post.php?action=edit&post=' . $new_post_id));
        exit;
    } else {
        // Caso o post original não seja encontrado, exibe uma mensagem de erro
        wp_die('Falha ao criar o post, não foi possível encontrar o post original: ' . $post_id);
    }
}
// Adiciona uma ação customizada para duplicar posts no admin
add_action('admin_action_custom_duplicate_post_as_draft', 'custom_duplicate_post_as_draft');

```
