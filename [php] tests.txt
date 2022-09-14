<?php

namespace App\Http\Controllers;

/**
 * Перед вами код контроллера по обновлению на сайте статей с тегами.
 * Проведите рефакторинг данного кода и предложите максимальное количество улучшений.
 */

use App\Models\Article;
use App\Models\Tag;
use Illuminate\Http\Request;
use Illuminate\Support\Carbon;

// Создать общий контроллер BaseController
// Вынести бизнес логику в класс сервиса или в класс действия (action) - последнее предпочтительнее
// Переработать создание и добавление тэгов (при прочтении кода не сразу понимаешь что делает код)
// Добавить транзакцию (статья + тэги)
// Создать кастомный класс реквеста для валидации данных запроса
// Добавить phpdoc-блоки и комментарии
// Выпилить фасады, сделать их как аругенты в update или передать через конструктор в сервисы

class ArticleController extends Controller
{
    public function update(Request $request, Article $article)
    {
        $this->authorize('update', Article::class);

        $article->title = $request->title;
        $article->description = $request->description;
        $article->body = $request->body;

        if ($request->published != null)
            $article->published_at = Carbon::now()->format('Y-m-d H:i:s');

        $article->save();

        $tagsToAttach = function ($tags, $model) {
            foreach ($tags as $tag => $name) {
                $tag = Tag::firstOrCreate(['name' => $name]);
                $tag->articles()->attach($model);
            }
        };

        $tagsToDetach = function ($tags, $model) {
            foreach ($tags as $tag => $name) {
                $tag = Tag::where('name', $name)->first();
                $tag->articles()->detach($model);
            }
        };

        if (!$article->tags->isEmpty()) {
            $oldTags = $article->tags->pluck('name')->diff($request->tags);
            $newTags = $request->tags->diff($article->tags->pluck('name'));
            $tagsToAttach($newTags, $article);
            $tagsToDetach($oldTags, $article);
        } else {
            $tagsToAttach($request->tags, $article);
        }

        logger()->info("Article {$article->title} was created");

        return response()->json(['success' => true, 'article_id' => $article->id]);
    }
}
